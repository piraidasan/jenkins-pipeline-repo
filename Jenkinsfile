pipeline {

    agent any

    environment {

        DOCKERHUB_USER = 'piraidasan'

        BACKEND_IMAGE = 'piraidasan/etmsys-backend'
        FRONTEND_IMAGE = 'piraidasan/etmsys-frontend'

        DOCKER_CREDENTIALS = 'dockerhub-creds'
    }

    stages {

        /*
         * =========================================================
         * 1. CHECK TOOLS
         * =========================================================
         */

        stage('Check Tools') {

            steps {

                sh '''
                    set -e

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
                '''
            }
        }


        /*
         * =========================================================
         * 2. CHECKOUT BACKEND
         * =========================================================
         */

        stage('Checkout Backend') {

            steps {

                dir('backend') {

                    git(
                        branch: 'main',
                        credentialsId: 'github-creds',
                        url: 'https://github.com/piraidasan/TMS-backend.git'
                    )
                }
            }
        }


        /*
         * =========================================================
         * 3. CHECKOUT FRONTEND
         * =========================================================
         */

        stage('Checkout Frontend') {

            steps {

                dir('frontend') {

                    git(
                        branch: 'main',
                        credentialsId: 'github-creds',
                        url: 'https://github.com/piraidasan/TMS-Frontend.git'
                    )
                }
            }
        }


        /*
         * =========================================================
         * 4. BUILD BACKEND
         * =========================================================
         */

        stage('Build Backend') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        echo "===== Backend Build ====="

                        java -version
                        mvn -version

                        mvn clean package -DskipTests

                        echo "===== Backend JAR ====="

                        ls -lh target/*.jar
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 5. BUILD FRONTEND
         * =========================================================
         */

        stage('Build Frontend') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        echo "===== Frontend Dependencies ====="

                        npm ci

                        echo "===== Angular Production Build ====="

                        npm run build

                        echo "===== Frontend Output ====="

                        ls -lah dist/tms-frontend
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 6. BUILD BACKEND DOCKER IMAGE
         * =========================================================
         */

        stage('Build Backend Docker Image') {

            steps {

                dir('backend') {

                    sh '''
                        set -e

                        docker build \
                            -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                            -t ${BACKEND_IMAGE}:latest \
                            .

                        docker images | grep etmsys-backend
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 7. BUILD FRONTEND DOCKER IMAGE
         * =========================================================
         */

        stage('Build Frontend Docker Image') {

            steps {

                dir('frontend') {

                    sh '''
                        set -e

                        docker build \
                            -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                            -t ${FRONTEND_IMAGE}:latest \
                            .

                        docker images | grep etmsys-frontend
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 8. DOCKER HUB LOGIN
         * =========================================================
         */

        stage('Docker Hub Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
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


        /*
         * =========================================================
         * 9. PUSH BACKEND
         * =========================================================
         */

        stage('Push Backend Image') {

            steps {

                sh '''
                    set -e

                    docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    docker push ${BACKEND_IMAGE}:latest
                '''
            }
        }


        /*
         * =========================================================
         * 10. PUSH FRONTEND
         * =========================================================
         */

        stage('Push Frontend Image') {

            steps {

                sh '''
                    set -e

                    docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    docker push ${FRONTEND_IMAGE}:latest
                '''
            }
        }


        /*
         * =========================================================
         * 11. VERIFY IMAGES
         * =========================================================
         */

        stage('Verify Docker Images') {

            steps {

                sh '''
                    set -e

                    echo "===== Backend ====="

                    docker image inspect \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    echo "===== Frontend ====="

                    docker image inspect \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo "===== Images ====="

                    docker images | grep etmsys
                '''
            }
        }
    }


    /*
     * =============================================================
     * POST ACTIONS
     * =============================================================
     */

    post {

        success {

            echo '''
            ==========================================
              ETMSYS CI/CD BUILD SUCCESS
            ==========================================
            '''

            echo "Backend Image:"
            echo "${BACKEND_IMAGE}:${BUILD_NUMBER}"

            echo "Frontend Image:"
            echo "${FRONTEND_IMAGE}:${BUILD_NUMBER}"
        }


        failure {

            echo '''
            ==========================================
              ETMSYS CI/CD BUILD FAILED
            ==========================================
            '''
        }


        always {

            sh '''
                docker logout || true
            '''
        }
    }
}