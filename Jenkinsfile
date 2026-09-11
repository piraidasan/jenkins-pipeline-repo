pipeline {

    agent any

    environment {

        // =====================================================
        // APPLICATION
        // =====================================================

        BACKEND_IMAGE  = 'piraidasan/etmsys-backend'
        FRONTEND_IMAGE = 'piraidasan/etmsys-frontend'

        BACKEND_PORT  = '9093'
        FRONTEND_PORT = '30081'

        K8S_NAMESPACE = 'etmsys'

        // Kubernetes node used for external access
        K8S_NODE_IP = '172.16.1.89'


        // =====================================================
        // GITHUB
        // =====================================================

        GITHUB_CREDENTIALS = 'github-creds'


        // =====================================================
        // DOCKER HUB
        // =====================================================

        DOCKER_CREDENTIALS = 'dockerhub-creds'


        // =====================================================
        // DATABASE
        // =====================================================

        DB_HOST = '172.16.1.89'
        DB_PORT = '3306'
        DB_NAME = 'etmsys'


        // =====================================================
        // JAVA
        // =====================================================

        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk'


        // =====================================================
        // KUBERNETES
        // =====================================================

        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }


    options {

        timestamps()

        disableConcurrentBuilds()

        timeout(
            time: 45,
            unit: 'MINUTES'
        )

        buildDiscarder(
            logRotator(
                numToKeepStr: '20'
            )
        )
    }


    stages {


        // =====================================================
        // 1. CHECK TOOLS
        // =====================================================

        stage('Check Tools') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "CHECKING BUILD ENVIRONMENT"
                    echo "=========================================="

                    whoami
                    id

                    echo ""
                    echo "Git:"
                    git --version

                    echo ""
                    echo "Java:"
                    export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
                    export PATH="$JAVA_HOME/bin:$PATH"
                    java -version

                    echo ""
                    echo "Maven:"
                    mvn -version

                    echo ""
                    echo "Node:"
                    node --version

                    echo ""
                    echo "NPM:"
                    npm --version

                    echo ""
                    echo "Docker:"
                    docker --version

                    echo ""
                    echo "Kubectl:"
                    kubectl version --client

                    echo ""
                    echo "Docker access:"
                    docker ps

                    echo ""
                    echo "Kubernetes access:"
                    kubectl get nodes

                    echo ""
                    echo "=========================================="
                    echo "TOOL CHECK SUCCESS"
                    echo "=========================================="
                '''
            }
        }


        // =====================================================
        // 2. CHECK DATABASE
        // =====================================================

        stage('Check MySQL') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "CHECK MYSQL CONNECTIVITY"
                    echo "=========================================="

                    timeout 5 bash -c \
                        "</dev/tcp/${DB_HOST}/${DB_PORT}"

                    echo ""
                    echo "MySQL ${DB_HOST}:${DB_PORT} reachable."

                    echo ""
                    echo "=========================================="
                    echo "MYSQL CHECK SUCCESS"
                    echo "=========================================="
                '''
            }
        }


        // =====================================================
        // 3. CHECKOUT BACKEND
        // =====================================================

        stage('Checkout Backend') {

            steps {

                dir('backend') {

                    deleteDir()

                    git(
                        branch: 'main',
                        credentialsId: "${GITHUB_CREDENTIALS}",
                        url: 'https://github.com/piraidasan/TMS-backend.git'
                    )
                }
            }
        }


        // =====================================================
        // 4. CHECKOUT FRONTEND
        // =====================================================

        stage('Checkout Frontend') {

            steps {

                dir('frontend') {

                    deleteDir()

                    git(
                        branch: 'main',
                        credentialsId: "${GITHUB_CREDENTIALS}",
                        url: 'https://github.com/piraidasan/TMS-Frontend.git'
                    )
                }
            }
        }


        // =====================================================
        // 5. BACKEND TEST
        // =====================================================

        stage('Backend Build and Test') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
                        export PATH="$JAVA_HOME/bin:$PATH"

                        echo "=========================================="
                        echo "BACKEND BUILD"
                        echo "=========================================="

                        mvn clean test package

                        echo ""
                        echo "Backend JAR:"
                        ls -lh target/*.jar

                        echo ""
                        echo "BACKEND BUILD SUCCESS"
                    '''
                }
            }
        }


        // =====================================================
        // 6. FRONTEND TEST
        // =====================================================

        stage('Frontend Build and Test') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "FRONTEND BUILD"
                        echo "=========================================="

                        npm ci

                        npm run build

                        echo ""
                        echo "Frontend dist:"
                        find dist -maxdepth 3 -type f | head -100

                        echo ""
                        echo "FRONTEND BUILD SUCCESS"
                    '''
                }
            }
        }


        // =====================================================
        // 7. BUILD DOCKER IMAGES
        // =====================================================

        stage('Build Docker Images') {

            parallel {

                stage('Build Backend Image') {

                    steps {

                        dir('backend') {

                            sh '''
                                set -e

                                echo "=========================================="
                                echo "BUILD BACKEND DOCKER IMAGE"
                                echo "=========================================="

                                docker build \
                                    --pull \
                                    -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                                    -t ${BACKEND_IMAGE}:latest \
                                    .

                                docker image inspect \
                                    ${BACKEND_IMAGE}:${BUILD_NUMBER}

                                echo ""
                                echo "Backend image:"
                                echo "${BACKEND_IMAGE}:${BUILD_NUMBER}"
                            '''
                        }
                    }
                }


                stage('Build Frontend Image') {

                    steps {

                        dir('frontend') {

                            sh '''
                                set -e

                                echo "=========================================="
                                echo "BUILD FRONTEND DOCKER IMAGE"
                                echo "=========================================="

                                docker build \
                                    --pull \
                                    -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                                    -t ${FRONTEND_IMAGE}:latest \
                                    .

                                docker image inspect \
                                    ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                                echo ""
                                echo "Frontend image:"
                                echo "${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                            '''
                        }
                    }
                }
            }
        }


        // =====================================================
        // 8. TEST BACKEND CONTAINER
        // =====================================================

        stage('Test Backend Container') {

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

                        echo "=========================================="
                        echo "TEST BACKEND CONTAINER"
                        echo "=========================================="

                        docker rm -f etmsys-backend-test 2>/dev/null || true

                        docker run -d \
                            --name etmsys-backend-test \
                            -p 19093:9093 \
                            -e SPRING_DATASOURCE_URL="jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" \
                            -e SPRING_DATASOURCE_USERNAME="${MYSQL_USERNAME}" \
                            -e SPRING_DATASOURCE_PASSWORD="${MYSQL_PASSWORD}" \
                            -e SERVER_PORT=9093 \
                            ${BACKEND_IMAGE}:${BUILD_NUMBER}

                        echo ""
                        echo "Waiting for backend..."

                        READY=false

                        for i in $(seq 1 30)
                        do

                            if curl -fsS \
                                http://127.0.0.1:19093/etmsys/v1/user/captcha \
                                > /tmp/backend-test.json
                            then

                                echo ""
                                echo "Backend API is responding."

                                cat /tmp/backend-test.json

                                READY=true

                                break

                            fi

                            if ! docker inspect \
                                -f '{{.State.Running}}' \
                                etmsys-backend-test 2>/dev/null |
                                grep -q true
                            then

                                echo ""
                                echo "Backend container stopped."

                                break
                            fi

                            echo "Waiting... ${i}/30"

                            sleep 5

                        done


                        if [ "$READY" != "true" ]
                        then

                            echo ""
                            echo "BACKEND CONTAINER TEST FAILED"

                            docker ps -a \
                                --filter name=etmsys-backend-test

                            docker logs \
                                --tail 200 \
                                etmsys-backend-test || true

                            exit 1
                        fi


                        docker rm -f etmsys-backend-test

                        echo ""
                        echo "BACKEND CONTAINER TEST SUCCESS"
                    '''
                }
            }
        }


        // =====================================================
        // 9. TEST FRONTEND CONTAINER
        // =====================================================

        stage('Test Frontend Container') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "TEST FRONTEND CONTAINER"
                    echo "=========================================="

                    docker rm -f etmsys-frontend-test 2>/dev/null || true

                    docker run -d \
                        --name etmsys-frontend-test \
                        -p 18081:80 \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    sleep 5

                    echo ""
                    echo "Testing NGINX configuration..."

                    docker exec \
                        etmsys-frontend-test \
                        nginx -t

                    echo ""
                    echo "Testing frontend..."

                    curl \
                        --fail \
                        --show-error \
                        --retry 10 \
                        --retry-delay 2 \
                        --retry-connrefused \
                        http://127.0.0.1:18081/

                    echo ""
                    echo "Frontend container test SUCCESS"

                    docker rm -f etmsys-frontend-test
                '''
            }
        }


        // =====================================================
        // 10. DOCKER HUB LOGIN
        // =====================================================

        stage('Docker Hub Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "$DOCKER_PASSWORD" |
                        docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                    '''
                }
            }
        }


        // =====================================================
        // 11. PUSH IMAGES
        // =====================================================

        stage('Push Docker Images') {

            parallel {

                stage('Push Backend') {

                    steps {

                        sh '''
                            set -e

                            docker push \
                                ${BACKEND_IMAGE}:${BUILD_NUMBER}

                            docker push \
                                ${BACKEND_IMAGE}:latest
                        '''
                    }
                }


                stage('Push Frontend') {

                    steps {

                        sh '''
                            set -e

                            docker push \
                                ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                            docker push \
                                ${FRONTEND_IMAGE}:latest
                        '''
                    }
                }
            }
        }


        // =====================================================
        // 12. PREPARE KUBERNETES
        // =====================================================

        stage('Prepare Kubernetes') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "KUBERNETES PREPARATION"
                    echo "=========================================="

                    kubectl get nodes

                    kubectl apply \
                        -f k8s/namespace.yaml

                    echo ""
                    echo "Namespace:"
                    kubectl get namespace ${K8S_NAMESPACE}
                '''
            }
        }


        // =====================================================
        // 13. CREATE DB SECRET FROM JENKINS
        // =====================================================

        stage('Configure Database Secret') {

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

                        echo "=========================================="
                        echo "CONFIGURE KUBERNETES DB SECRET"
                        echo "=========================================="

                        kubectl -n ${K8S_NAMESPACE} create secret generic etmsys-db-secret \
                            --from-literal=username="${MYSQL_USERNAME}" \
                            --from-literal=password="${MYSQL_PASSWORD}" \
                            --dry-run=client \
                            -o yaml |
                        kubectl apply -f -

                        echo "Database secret updated."
                    '''
                }
            }
        }


        // =====================================================
        // 14. DEPLOY KUBERNETES
        // =====================================================

        stage('Deploy Kubernetes') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "DEPLOY APPLICATION TO KUBERNETES"
                    echo "=========================================="

                    kubectl apply \
                        -f k8s/backend-deployment.yaml

                    kubectl apply \
                        -f k8s/backend-service.yaml

                    kubectl apply \
                        -f k8s/frontend-deployment.yaml

                    kubectl apply \
                        -f k8s/frontend-service.yaml


                    echo ""
                    echo "Updating backend image..."

                    kubectl -n ${K8S_NAMESPACE} \
                        set image deployment/etmsys-backend \
                        backend=${BACKEND_IMAGE}:${BUILD_NUMBER}


                    echo ""
                    echo "Updating frontend image..."

                    kubectl -n ${K8S_NAMESPACE} \
                        set image deployment/etmsys-frontend \
                        frontend=${FRONTEND_IMAGE}:${BUILD_NUMBER}


                    echo ""
                    echo "Kubernetes deployment updated."
                '''
            }
        }


        // =====================================================
        // 15. WAIT BACKEND
        // =====================================================

        stage('Verify Backend Deployment') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "VERIFY BACKEND DEPLOYMENT"
                    echo "=========================================="

                    kubectl -n ${K8S_NAMESPACE} \
                        rollout status \
                        deployment/etmsys-backend \
                        --timeout=5m


                    echo ""
                    echo "Backend pods:"

                    kubectl -n ${K8S_NAMESPACE} \
                        get pods \
                        -l app=etmsys-backend \
                        -o wide


                    echo ""
                    echo "Backend service:"

                    kubectl -n ${K8S_NAMESPACE} \
                        get svc etmsys-backend
                '''
            }
        }


        // =====================================================
        // 16. WAIT FRONTEND
        // =====================================================

        stage('Verify Frontend Deployment') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "VERIFY FRONTEND DEPLOYMENT"
                    echo "=========================================="

                    kubectl -n ${K8S_NAMESPACE} \
                        rollout status \
                        deployment/etmsys-frontend \
                        --timeout=5m


                    echo ""
                    echo "Frontend pods:"

                    kubectl -n ${K8S_NAMESPACE} \
                        get pods \
                        -l app=etmsys-frontend \
                        -o wide


                    echo ""
                    echo "Frontend service:"

                    kubectl -n ${K8S_NAMESPACE} \
                        get svc etmsys-frontend
                '''
            }
        }


        // =====================================================
        // 17. APPLICATION HEALTH CHECK
        // =====================================================

        stage('Application Health Check') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "APPLICATION HEALTH CHECK"
                    echo "=========================================="


                    echo ""
                    echo "Frontend URL:"
                    echo "http://${K8S_NODE_IP}:${FRONTEND_PORT}/"


                    echo ""
                    echo "Testing frontend..."

                    curl \
                        --fail \
                        --show-error \
                        --retry 10 \
                        --retry-delay 3 \
                        --retry-connrefused \
                        http://${K8S_NODE_IP}:${FRONTEND_PORT}/


                    echo ""
                    echo "Testing CAPTCHA through frontend NGINX..."

                    curl \
                        --fail \
                        --show-error \
                        --retry 10 \
                        --retry-delay 3 \
                        --retry-connrefused \
                        http://${K8S_NODE_IP}:${FRONTEND_PORT}/etmsys/v1/user/captcha \
                        > /tmp/captcha-response.json


                    echo ""
                    echo "CAPTCHA response:"

                    cat /tmp/captcha-response.json


                    echo ""
                    echo "=========================================="
                    echo "APPLICATION HEALTH CHECK SUCCESS"
                    echo "=========================================="
                '''
            }
        }


        // =====================================================
        // 18. FINAL STATUS
        // =====================================================

        stage('Final Kubernetes Status') {

            steps {

                sh '''
                    echo ""
                    echo "=========================================="
                    echo "FINAL KUBERNETES STATUS"
                    echo "=========================================="

                    kubectl -n ${K8S_NAMESPACE} get pods -o wide

                    echo ""
                    kubectl -n ${K8S_NAMESPACE} get svc

                    echo ""
                    kubectl -n ${K8S_NAMESPACE} get deployments

                    echo ""
                    echo "=========================================="
                    echo "CI/CD DEPLOYMENT SUCCESS"
                    echo "=========================================="

                    echo ""
                    echo "Application:"
                    echo "http://${K8S_NODE_IP}:${FRONTEND_PORT}/login"
                '''
            }
        }
    }


    // =========================================================
    // POST ACTIONS
    // =========================================================

    post {

        always {

            sh '''
                docker rm -f \
                    etmsys-backend-test \
                    etmsys-frontend-test \
                    2>/dev/null || true
            '''

            sh '''
                docker logout || true
            '''
        }


        success {

            echo """
==========================================
ETMSYS CI/CD SUCCESS
==========================================

Frontend:
http://${K8S_NODE_IP}:${FRONTEND_PORT}/login

Backend:
Kubernetes internal service
etmsys-backend:9093

Docker Images:
${BACKEND_IMAGE}:${BUILD_NUMBER}
${FRONTEND_IMAGE}:${BUILD_NUMBER}

Kubernetes Namespace:
${K8S_NAMESPACE}

==========================================
"""
        }


        failure {

            echo """
==========================================
ETMSYS CI/CD FAILED
==========================================

Checking Kubernetes status...
==========================================
"""

            sh '''
                kubectl -n ${K8S_NAMESPACE} get pods -o wide || true

                echo ""

                kubectl -n ${K8S_NAMESPACE} get deployments || true

                echo ""

                kubectl -n ${K8S_NAMESPACE} get events \
                    --sort-by=.lastTimestamp \
                    | tail -50 || true
            '''
        }
    }
}
