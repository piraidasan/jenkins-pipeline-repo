pipeline {

    agent any

    environment {

        BACKEND_IMAGE  = 'piraidasan/etmsys-backend'
        FRONTEND_IMAGE = 'piraidasan/etmsys-frontend'

        DOCKER_CREDENTIALS = 'dockerhub-creds'
        GITHUB_CREDENTIALS = 'github-creds'

        // Current Docker-host environment
        BACKEND_PORT = '9093'
        FRONTEND_PORT = '8081'

        // Current MySQL server
        DB_HOST = '172.16.1.89'
        DB_PORT = '3306'
        DB_NAME = 'etmsys'
        DB_USER = 'javi'
    }

    stages {

        stage('Check Tools') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "ETMSYS CI/CD - CHECK TOOLS"
                    echo "========================================"

                    echo "===== Git ====="
                    git --version

                    echo "===== Java ====="
                    java -version

                    echo "===== Maven ====="
                    mvn -version

                    echo "===== Node ====="
                    node --version

                    echo "===== npm ====="
                    npm --version

                    echo "===== Docker ====="
                    docker version

                    echo "========================================"
                '''
            }
        }


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


        stage('Build Backend') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        echo "========================================"
                        echo "BUILDING ETMSYS BACKEND"
                        echo "========================================"

                        java -version
                        mvn -version

                        mvn clean package -DskipTests

                        echo "===== Backend JAR ====="

                        ls -lh target/*.jar
                    '''
                }
            }
        }


        stage('Build Frontend') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        echo "========================================"
                        echo "BUILDING ETMSYS FRONTEND"
                        echo "========================================"

                        npm ci

                        npm run build

                        echo "===== Frontend Output ====="

                        ls -lah dist/tms-frontend
                    '''
                }
            }
        }


        stage('Build Backend Docker Image') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        echo "========================================"
                        echo "BUILD BACKEND DOCKER IMAGE"
                        echo "========================================"

                        docker build \
                            -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                            -t ${BACKEND_IMAGE}:latest \
                            .

                        echo "===== Backend Images ====="

                        docker images | grep etmsys-backend
                    '''
                }
            }
        }


        stage('Build Frontend Docker Image') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        echo "========================================"
                        echo "BUILD FRONTEND DOCKER IMAGE"
                        echo "========================================"

                        docker build \
                            -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                            -t ${FRONTEND_IMAGE}:latest \
                            .

                        echo "===== Frontend Images ====="

                        docker images | grep etmsys-frontend
                    '''
                }
            }
        }


        stage('Test Backend Container') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "TEST BACKEND DOCKER IMAGE"
                    echo "========================================"

                    docker rm -f etmsys-backend-test 2>/dev/null || true

                    docker run -d \
                        --name etmsys-backend-test \
                        -p 19093:9093 \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    echo "Waiting for backend..."

                    sleep 15

                    docker ps | grep etmsys-backend-test

                    echo "===== Backend Logs ====="

                    docker logs --tail 50 etmsys-backend-test

                    echo "===== Backend Test ====="

                    curl -f \
                        http://127.0.0.1:19093/etmsys/v1/user/captcha

                    echo

                    docker rm -f etmsys-backend-test
                '''
            }
        }


        stage('Test Frontend Container') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "TEST FRONTEND DOCKER IMAGE"
                    echo "========================================"

                    docker rm -f etmsys-frontend-test 2>/dev/null || true

                    docker run -d \
                        --name etmsys-frontend-test \
                        -p 18081:80 \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo "Waiting for frontend..."

                    sleep 5

                    docker ps | grep etmsys-frontend-test

                    echo "===== NGINX CONFIG TEST ====="

                    docker exec \
                        etmsys-frontend-test \
                        nginx -t

                    echo "===== Frontend HTTP Test ====="

                    curl -f http://127.0.0.1:18081/

                    echo

                    docker rm -f etmsys-frontend-test
                '''
            }
        }


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

                        echo "$DOCKER_PASSWORD" | \
                            docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                    '''
                }
            }
        }


        stage('Push Backend Image') {

            steps {

                sh '''
                    set -e

                    echo "Pushing backend:"

                    docker push \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }


        stage('Push Frontend Image') {

            steps {

                sh '''
                    set -e

                    echo "Pushing frontend:"

                    docker push \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }


        stage('Push Latest Tags') {

            steps {

                sh '''
                    set -e

                    docker push \
                        ${BACKEND_IMAGE}:latest

                    docker push \
                        ${FRONTEND_IMAGE}:latest
                '''
            }
        }


        stage('Verify Images') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "VERIFY DOCKER IMAGES"
                    echo "========================================"

                    docker image inspect \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    docker image inspect \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo "===== ETMSYS Images ====="

                    docker images | grep etmsys
                '''
            }
        }
    }


    post {

        success {

            echo """
            ==========================================
              ETMSYS CI/CD BUILD SUCCESS
            ==========================================

            Jenkins Build:
            ${BUILD_NUMBER}

            Backend:
            ${BACKEND_IMAGE}:${BUILD_NUMBER}

            Frontend:
            ${FRONTEND_IMAGE}:${BUILD_NUMBER}

            Backend Port:
            ${BACKEND_PORT}

            Frontend Port:
            ${FRONTEND_PORT}

            MySQL:
            ${DB_HOST}:${DB_PORT}

            Database:
            ${DB_NAME}

            ==========================================
            """
        }


        failure {

            echo """
            ==========================================
              ETMSYS CI/CD BUILD FAILED
            ==========================================

            Check the failed Jenkins stage.

            ==========================================
            """
        }


        always {

            sh '''
                docker rm -f etmsys-backend-test 2>/dev/null || true
                docker rm -f etmsys-frontend-test 2>/dev/null || true

                docker logout || true
            '''
        }
    }
}