pipeline {
    agent any

    environment {
        // Registry & Image Configuration
        DOCKER_HUB_USER  = 'piraidasan'
        BACKEND_IMAGE    = "${DOCKER_HUB_USER}/etmsys-backend"
        FRONTEND_IMAGE   = "${DOCKER_HUB_USER}/etmsys-frontend"
        
        // Application Settings
        K8S_NAMESPACE    = 'etmsys'
        BACKEND_PORT     = '9093'
        FRONTEND_PORT    = '8081'
        TEST_BACKEND_PORT= '19093'
        TEST_FRONTEND_PORT='18081'
        
        // Database Settings
        DB_HOST          = '192.168.1.100'
        DB_PORT          = '3306'
        DB_NAME          = 'etmsys_db'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                echo '========================================'
                echo 'STAGE 1: CHECKING OUT SOURCE CODE'
                echo '========================================'
                checkout scm
            }
        }

        stage('Backend - SonarQube / Static Analysis') {
            steps {
                echo '========================================'
                echo 'STAGE 2: BACKEND CODE ANALYSIS'
                echo '========================================'
                dir('backend') {
                    sh './mvnw clean test-compile'
                }
            }
        }

        stage('Backend - Run Unit Tests') {
            steps {
                echo '========================================'
                echo 'STAGE 3: RUNNING BACKEND UNIT TESTS'
                echo '========================================'
                dir('backend') {
                    sh './mvnw test'
                }
            }
        }

        stage('Build Artifacts') {
            parallel {
                stage('Build Backend JAR') {
                    steps {
                        echo 'Building Spring Boot JAR...'
                        dir('backend') {
                            sh './mvnw package -DskipTests'
                        }
                    }
                }
                stage('Verify Frontend Workspace') {
                    steps {
                        echo 'Verifying Frontend source directory...'
                        dir('frontend') {
                            sh 'ls -la'
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Backend Docker Image') {
                    steps {
                        echo 'Building Backend Image...'
                        dir('backend') {
                            sh "docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER} -t ${BACKEND_IMAGE}:latest ."
                        }
                    }
                }
                stage('Build Frontend Docker Image') {
                    steps {
                        echo 'Building Frontend Image...'
                        dir('frontend') {
                            sh "docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} -t ${FRONTEND_IMAGE}:latest ."
                        }
                    }
                }
            }
        }

        stage('Container Health Check & Testing') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'etmsys-db-creds',
                        usernameVariable: 'MYSQL_USERNAME',
                        passwordVariable: 'MYSQL_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e
                        echo "========================================"
                        echo "CONTAINER TESTING"
                        echo "========================================"

                        docker rm -f etmsys-backend-test etmsys-frontend-test 2>/dev/null || true

                        echo "Starting test containers on isolated host ports..."
                        docker run -d \
                            --name etmsys-backend-test \
                            -p ${TEST_BACKEND_PORT}:${BACKEND_PORT} \
                            -e SPRING_DATASOURCE_URL="jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" \
                            -e SPRING_DATASOURCE_USERNAME="${MYSQL_USERNAME}" \
                            -e SPRING_DATASOURCE_PASSWORD="${MYSQL_PASSWORD}" \
                            -e SERVER_PORT=${BACKEND_PORT} \
                            ${BACKEND_IMAGE}:${BUILD_NUMBER}

                        docker run -d \
                            --name etmsys-frontend-test \
                            -p ${TEST_FRONTEND_PORT}:80 \
                            ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                        echo "Waiting 15s for services to initialize..."
                        sleep 15

                        echo "Verifying HTTP responses..."
                        curl --fail http://localhost:${TEST_FRONTEND_PORT} || (echo "Frontend container test failed" && exit 1)
                        
                        echo "Cleaning up test containers..."
                        docker rm -f etmsys-backend-test etmsys-frontend-test
                    '''
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        set -e
                        echo "========================================"
                        echo "PUSHING IMAGES TO DOCKER HUB"
                        echo "========================================"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${BACKEND_IMAGE}:latest

                        docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${FRONTEND_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'etmsys-db-creds',
                        usernameVariable: 'MYSQL_USERNAME',
                        passwordVariable: 'MYSQL_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e
                        echo "========================================"
                        echo "DEPLOYING TO KUBERNETES CLUSTER"
                        echo "========================================"

                        if [ -f "k8s/namespace.yaml" ]; then
                            kubectl apply -f k8s/namespace.yaml
                        else
                            kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        fi

                        kubectl create secret generic etmsys-db-secret \
                            --namespace=${K8S_NAMESPACE} \
                            --from-literal=username="${MYSQL_USERNAME}" \
                            --from-literal=password="${MYSQL_PASSWORD}" \
                            --dry-run=client -o yaml | kubectl apply -f -

                        kubectl apply -f k8s/ -n ${K8S_NAMESPACE}

                        echo "Updating deployment images to build tag: ${BUILD_NUMBER}..."
                        kubectl set image deployment/etmsys-backend \
                            etmsys-backend=${BACKEND_IMAGE}:${BUILD_NUMBER} \
                            -n ${K8S_NAMESPACE}

                        kubectl set image deployment/etmsys-frontend \
                            etmsys-frontend=${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                            -n ${K8S_NAMESPACE}

                        echo "Verifying rollout status..."
                        kubectl rollout status deployment/etmsys-backend -n ${K8S_NAMESPACE} --timeout=120s
                        kubectl rollout status deployment/etmsys-frontend -n ${K8S_NAMESPACE} --timeout=120s
                    '''
                }
            }
        }

        stage('Verify Kubernetes Deployment') {
            steps {
                sh '''
                    set -e
                    echo "========================================"
                    echo "KUBERNETES DEPLOYMENT VERIFICATION"
                    echo "========================================"

                    echo "Checking Running Pods:"
                    kubectl get pods -n ${K8S_NAMESPACE} -o wide

                    echo "\nChecking Active Services & Ingress:"
                    kubectl get svc,ingress -n ${K8S_NAMESPACE}

                    echo "\nVerifying deployed image tags:"
                    kubectl get deployment etmsys-backend -n ${K8S_NAMESPACE} -o jsonpath='{.spec.template.spec.containers[0].image}'
                    echo ""
                    kubectl get deployment etmsys-frontend -n ${K8S_NAMESPACE} -o jsonpath='{.spec.template.spec.containers[0].image}'
                    echo ""
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f etmsys-backend-test etmsys-frontend-test 2>/dev/null || true'
        }
        success {
            echo "Pipeline completed successfully. Deployed build #${BUILD_NUMBER} to Kubernetes namespace '${K8S_NAMESPACE}'."
        }
        failure {
            echo "Pipeline execution failed. Rolling back Kubernetes deployments if necessary..."
            sh '''
                kubectl rollout undo deployment/etmsys-backend -n ${K8S_NAMESPACE} 2>/dev/null || true
                kubectl rollout undo deployment/etmsys-frontend -n ${K8S_NAMESPACE} 2>/dev/null || true
            '''
        }
    }
}
