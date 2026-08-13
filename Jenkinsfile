pipeline {

    agent any

    environment {

        /*
         * =========================================================
         * JAVA 21
         * =========================================================
         */
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk'

        /*
         * =========================================================
         * DOCKER HUB
         * =========================================================
         */
        BACKEND_IMAGE  = 'piraidasan/etmsys-backend'
        FRONTEND_IMAGE = 'piraidasan/etmsys-frontend'

        DOCKER_CREDENTIALS = 'dockerhub-creds'
        GITHUB_CREDENTIALS = 'github-creds'

        /*
         * =========================================================
         * APPLICATION PORTS
         * =========================================================
         */
        BACKEND_PORT  = '9093'
        FRONTEND_PORT = '8081'

        /*
         * =========================================================
         * MYSQL
         * =========================================================
         */
        DB_HOST = '172.16.1.89'
        DB_PORT = '3306'
        DB_NAME = 'etmsys'
        DB_USER = 'javi'
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

                    export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
                    export PATH="$JAVA_HOME/bin:$PATH"

                    echo "========================================"
                    echo "ETMSYS CI/CD - CHECK TOOLS"
                    echo "========================================"

                    echo ""
                    echo "===== USER ====="
                    whoami
                    id

                    echo ""
                    echo "===== Git ====="
                    git --version

                    echo ""
                    echo "===== JAVA_HOME ====="
                    echo "$JAVA_HOME"

                    echo ""
                    echo "===== Java ====="
                    which java
                    readlink -f $(which java)
                    java -version

                    echo ""
                    echo "===== Maven ====="
                    which mvn
                    mvn -version

                    echo ""
                    echo "===== Node ====="
                    node --version

                    echo ""
                    echo "===== npm ====="
                    npm --version

                    echo ""
                    echo "===== Docker ====="
                    docker version

                    echo ""
                    echo "===== Docker Access ====="
                    docker ps

                    echo ""
                    echo "========================================"
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

                    deleteDir()

                    git(
                        branch: 'main',
                        credentialsId: "${GITHUB_CREDENTIALS}",
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

                    deleteDir()

                    git(
                        branch: 'main',
                        credentialsId: "${GITHUB_CREDENTIALS}",
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

                        export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
                        export PATH="$JAVA_HOME/bin:$PATH"

                        echo "========================================"
                        echo "BUILDING ETMSYS BACKEND"
                        echo "========================================"

                        echo ""
                        echo "===== Java ====="
                        java -version

                        echo ""
                        echo "===== Maven ====="
                        mvn -version

                        echo ""
                        echo "===== Maven Build ====="

                        mvn clean package -DskipTests

                        echo ""
                        echo "===== Backend JAR ====="

                        ls -lh target/*.jar

                        echo ""
                        echo "Backend build completed successfully."
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

                        echo "========================================"
                        echo "BUILDING ETMSYS FRONTEND"
                        echo "========================================"

                        echo ""
                        echo "===== Node ====="
                        node --version

                        echo ""
                        echo "===== npm ====="
                        npm --version

                        echo ""
                        echo "===== Install Dependencies ====="

                        npm ci

                        echo ""
                        echo "===== Angular Production Build ====="

                        npm run build

                        echo ""
                        echo "===== Frontend Output ====="

                        find dist -maxdepth 3 -type f | head -100

                        echo ""
                        echo "===== dist directory ====="

                        ls -lah dist/

                        echo ""
                        echo "Frontend build completed successfully."
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

                        echo "========================================"
                        echo "BUILD BACKEND DOCKER IMAGE"
                        echo "========================================"

                        docker build \
                            -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                            -t ${BACKEND_IMAGE}:latest \
                            .

                        echo ""
                        echo "===== Backend Images ====="

                        docker images | grep etmsys-backend

                        echo ""
                        echo "Backend Docker image created:"
                        echo "${BACKEND_IMAGE}:${BUILD_NUMBER}"
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

                        echo "========================================"
                        echo "BUILD FRONTEND DOCKER IMAGE"
                        echo "========================================"

                        docker build \
                            -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                            -t ${FRONTEND_IMAGE}:latest \
                            .

                        echo ""
                        echo "===== Frontend Images ====="

                        docker images | grep etmsys-frontend

                        echo ""
                        echo "Frontend Docker image created:"
                        echo "${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 8. TEST BACKEND CONTAINER
         * =========================================================
         */

        stage('Test Backend Container') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "TEST BACKEND DOCKER IMAGE"
                    echo "========================================"

                    docker rm -f etmsys-backend-test 2>/dev/null || true

                    echo ""
                    echo "Starting backend test container..."

                    docker run -d \
                        --name etmsys-backend-test \
                        -p 19093:9093 \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    echo ""
                    echo "Waiting for backend application..."

                    sleep 20

                    echo ""
                    echo "===== Container ====="

                    docker ps -a | grep etmsys-backend-test

                    echo ""
                    echo "===== Backend Logs ====="

                    docker logs --tail 100 etmsys-backend-test

                    echo ""
                    echo "===== Backend API Test ====="

                    curl --fail --show-error \
                        --retry 5 \
                        --retry-delay 3 \
                        --retry-connrefused \
                        http://127.0.0.1:19093/etmsys/v1/user/captcha

                    echo ""
                    echo ""
                    echo "Backend container test SUCCESS."

                    docker rm -f etmsys-backend-test
                '''
            }
        }


        /*
         * =========================================================
         * 9. TEST FRONTEND CONTAINER
         * =========================================================
         */

        stage('Test Frontend Container') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "TEST FRONTEND DOCKER IMAGE"
                    echo "========================================"

                    docker rm -f etmsys-frontend-test 2>/dev/null || true

                    echo ""
                    echo "Starting frontend test container..."

                    docker run -d \
                        --name etmsys-frontend-test \
                        -p 18081:80 \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo ""
                    echo "Waiting for NGINX..."

                    sleep 5

                    echo ""
                    echo "===== Container ====="

                    docker ps -a | grep etmsys-frontend-test

                    echo ""
                    echo "===== NGINX Configuration ====="

                    docker exec \
                        etmsys-frontend-test \
                        nginx -t

                    echo ""
                    echo "===== Frontend HTTP Test ====="

                    curl --fail --show-error \
                        --retry 5 \
                        --retry-delay 2 \
                        --retry-connrefused \
                        http://127.0.0.1:18081/

                    echo ""
                    echo ""
                    echo "Frontend container test SUCCESS."

                    docker rm -f etmsys-frontend-test
                '''
            }
        }


        /*
         * =========================================================
         * 10. DOCKER HUB LOGIN
         * =========================================================
         */

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

                        echo "========================================"
                        echo "DOCKER HUB LOGIN"
                        echo "========================================"

                        echo "$DOCKER_PASSWORD" | \
                            docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin

                        echo "Docker Hub login successful."
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 11. PUSH BACKEND IMAGE
         * =========================================================
         */

        stage('Push Backend Image') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "PUSH BACKEND IMAGE"
                    echo "========================================"

                    docker push \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    echo ""
                    echo "Backend image pushed:"
                    echo "${BACKEND_IMAGE}:${BUILD_NUMBER}"
                '''
            }
        }


        /*
         * =========================================================
         * 12. PUSH FRONTEND IMAGE
         * =========================================================
         */

        stage('Push Frontend Image') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "PUSH FRONTEND IMAGE"
                    echo "========================================"

                    docker push \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo ""
                    echo "Frontend image pushed:"
                    echo "${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                '''
            }
        }


        /*
         * =========================================================
         * 13. PUSH LATEST
         * =========================================================
         */

        stage('Push Latest Tags') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "PUSH LATEST TAGS"
                    echo "========================================"

                    docker push \
                        ${BACKEND_IMAGE}:latest

                    docker push \
                        ${FRONTEND_IMAGE}:latest

                    echo ""
                    echo "Latest tags pushed successfully."
                '''
            }
        }


        /*
         * =========================================================
         * 14. VERIFY IMAGES
         * =========================================================
         */

        stage('Verify Images') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "VERIFY DOCKER IMAGES"
                    echo "========================================"

                    echo ""
                    echo "===== Backend ====="

                    docker image inspect \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    echo ""
                    echo "===== Frontend ====="

                    docker image inspect \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo ""
                    echo "===== ETMSYS Images ====="

                    docker images | grep etmsys

                    echo ""
                    echo "========================================"
                    echo "IMAGE VERIFICATION SUCCESS"
                    echo "========================================"
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

            echo """
            ==========================================
                 ETMSYS CI/CD BUILD SUCCESS
            ==========================================

            Jenkins Build:
            ${BUILD_NUMBER}

            Backend Image:
            ${BACKEND_IMAGE}:${BUILD_NUMBER}

            Frontend Image:
            ${FRONTEND_IMAGE}:${BUILD_NUMBER}

            Backend Latest:
            ${BACKEND_IMAGE}:latest

            Frontend Latest:
            ${FRONTEND_IMAGE}:latest

            Application Backend:
            http://172.16.1.89:${BACKEND_PORT}

            Application Frontend:
            http://172.16.1.89:${FRONTEND_PORT}

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

            Jenkins Build:
            ${BUILD_NUMBER}

            Check the failed stage in Jenkins Console Output.

            ==========================================
            """
        }


        always {

            sh '''
                echo "Cleaning temporary test containers..."

                docker rm -f etmsys-backend-test 2>/dev/null || true
                docker rm -f etmsys-frontend-test 2>/dev/null || true

                echo "Logging out from Docker Hub..."

                docker logout || true
            '''
        }
    }
}