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
         * MYSQL DATABASE
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
                    echo "TOOL CHECK SUCCESS"
                    echo "========================================"
                '''
            }
        }


        /*
         * =========================================================
         * 2. CHECK MYSQL CONNECTIVITY
         * =========================================================
         */

        stage('Check MySQL Connectivity') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "CHECK MYSQL CONNECTIVITY"
                    echo "========================================"

                    echo ""
                    echo "Database Host : ${DB_HOST}"
                    echo "Database Port : ${DB_PORT}"
                    echo "Database Name : ${DB_NAME}"
                    echo "Database User : ${DB_USER}"

                    echo ""
                    echo "Checking TCP connectivity..."

                    timeout 5 bash -c \
                        "</dev/tcp/${DB_HOST}/${DB_PORT}"

                    echo ""
                    echo "MySQL TCP port is reachable."

                    echo ""
                    echo "========================================"
                    echo "MYSQL CONNECTIVITY SUCCESS"
                    echo "========================================"
                '''
            }
        }


        /*
         * =========================================================
         * 3. CHECKOUT BACKEND
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
         * 4. CHECKOUT FRONTEND
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
         * 5. BUILD BACKEND
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

                        echo ""
                        echo "========================================"
                        echo "BACKEND BUILD SUCCESS"
                        echo "========================================"
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 6. BUILD FRONTEND
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

                        echo ""
                        echo "========================================"
                        echo "FRONTEND BUILD SUCCESS"
                        echo "========================================"
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 7. BUILD BACKEND DOCKER IMAGE
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

                        echo ""
                        echo "========================================"
                        echo "BACKEND DOCKER BUILD SUCCESS"
                        echo "========================================"
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 8. BUILD FRONTEND DOCKER IMAGE
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

                        echo ""
                        echo "========================================"
                        echo "FRONTEND DOCKER BUILD SUCCESS"
                        echo "========================================"
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 9. TEST BACKEND DOCKER CONTAINER
         * =========================================================
         */

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

                        echo "========================================"
                        echo "TEST BACKEND DOCKER IMAGE"
                        echo "========================================"

                        docker rm -f etmsys-backend-test 2>/dev/null || true

                        echo ""
                        echo "Starting backend test container..."

                        docker run -d \
                            --name etmsys-backend-test \
                            -p 19093:9093 \
                            -e SPRING_DATASOURCE_URL="jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" \
                            -e SPRING_DATASOURCE_USERNAME="${MYSQL_USERNAME}" \
                            -e SPRING_DATASOURCE_PASSWORD="${MYSQL_PASSWORD}" \
                            -e SERVER_PORT=9093 \
                            ${BACKEND_IMAGE}:${BUILD_NUMBER}

                        echo ""
                        echo "Backend container started."

                        echo ""
                        echo "===== Container ID ====="

                        docker ps -a \
                            --filter name=etmsys-backend-test

                        echo ""
                        echo "Waiting for Spring Boot application..."

                        BACKEND_READY=false

                        for i in $(seq 1 30)
                        do

                            echo "Health check attempt ${i}/30..."

                            if curl -s \
                                --fail \
                                http://127.0.0.1:19093/etmsys/v1/user/captcha \
                                > /tmp/backend-response.txt 2>/dev/null
                            then

                                echo ""
                                echo "Backend API is responding."

                                BACKEND_READY=true

                                break
                            fi

                            if ! docker inspect \
                                -f '{{.State.Running}}' \
                                etmsys-backend-test 2>/dev/null | grep -q true
                            then

                                echo ""
                                echo "Backend container has stopped."

                                break
                            fi

                            sleep 5

                        done


                        echo ""
                        echo "========================================"
                        echo "BACKEND CONTAINER STATUS"
                        echo "========================================"

                        docker ps -a \
                            --filter name=etmsys-backend-test


                        echo ""
                        echo "========================================"
                        echo "BACKEND CONTAINER LOGS"
                        echo "========================================"

                        docker logs \
                            --tail 200 \
                            etmsys-backend-test || true


                        if [ "$BACKEND_READY" != "true" ]
                        then

                            echo ""
                            echo "========================================"
                            echo "BACKEND TEST FAILED"
                            echo "========================================"

                            echo ""
                            echo "Possible causes:"
                            echo "1. MySQL connection failure"
                            echo "2. Wrong DB username/password"
                            echo "3. Spring Boot configuration failure"
                            echo "4. Database schema problem"
                            echo "5. Application startup exception"
                            echo "6. Port 9093 configuration problem"

                            echo ""
                            echo "Inspect complete container logs above."

                            exit 1

                        fi


                        echo ""
                        echo "========================================"
                        echo "BACKEND API TEST SUCCESS"
                        echo "========================================"

                        echo ""
                        echo "Endpoint:"
                        echo "http://127.0.0.1:19093/etmsys/v1/user/captcha"


                        echo ""
                        echo "Backend container test SUCCESS."

                        docker rm -f etmsys-backend-test

                    '''
                }
            }
        }


        /*
         * =========================================================
         * 10. TEST FRONTEND DOCKER CONTAINER
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

                    docker ps -a \
                        --filter name=etmsys-frontend-test


                    echo ""
                    echo "===== NGINX Configuration ====="

                    docker exec \
                        etmsys-frontend-test \
                        nginx -t


                    echo ""
                    echo "===== Frontend HTTP Test ====="

                    curl --fail \
                        --show-error \
                        --retry 10 \
                        --retry-delay 2 \
                        --retry-connrefused \
                        http://127.0.0.1:18081/


                    echo ""
                    echo "Frontend container test SUCCESS."

                    docker rm -f etmsys-frontend-test


                    echo ""
                    echo "========================================"
                    echo "FRONTEND CONTAINER TEST SUCCESS"
                    echo "========================================"
                '''
            }
        }


        /*
         * =========================================================
         * 11. DOCKER HUB LOGIN
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

                        echo ""
                        echo "Docker Hub login successful."
                    '''
                }
            }
        }


        /*
         * =========================================================
         * 12. PUSH BACKEND IMAGE
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
         * 13. PUSH FRONTEND IMAGE
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
         * 14. PUSH LATEST TAGS
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
         * 15. VERIFY IMAGES
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
                    echo "===== Backend Image ====="

                    docker images \
                        ${BACKEND_IMAGE}


                    echo ""
                    echo "===== Frontend Image ====="

                    docker images \
                        ${FRONTEND_IMAGE}


                    echo ""
                    echo "========================================"
                    echo "IMAGE VERIFICATION SUCCESS"
                    echo "========================================"
                '''
            }
        }


        /*
         * =========================================================
         * 16. DEPLOY APPLICATIONS
         * =========================================================
         */

        stage('Deploy Applications') {

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
                        echo "ETMSYS PRODUCTION DEPLOYMENT"
                        echo "========================================"

                        OLD_BACKEND_IMAGE="etmsys-backend:v1"
                        OLD_FRONTEND_IMAGE="etmsys-frontend:v4"

                        NEW_BACKEND_IMAGE="${BACKEND_IMAGE}:${BUILD_NUMBER}"
                        NEW_FRONTEND_IMAGE="${FRONTEND_IMAGE}:${BUILD_NUMBER}"

                        echo ""
                        echo "New Backend:"
                        echo "$NEW_BACKEND_IMAGE"

                        echo "New Frontend:"
                        echo "$NEW_FRONTEND_IMAGE"

                        echo ""
                        echo "========================================"
                        echo "STOPPING CURRENT APPLICATION"
                        echo "========================================"

                        docker stop backend 2>/dev/null || true
                        docker stop frontend 2>/dev/null || true

                        docker rm backend 2>/dev/null || true
                        docker rm frontend 2>/dev/null || true

                        echo ""
                        echo "========================================"
                        echo "STARTING NEW BACKEND"
                        echo "========================================"

                        docker run -d \
                            --name backend \
                            --restart always \
                            -p ${BACKEND_PORT}:9093 \
                            -e SPRING_DATASOURCE_URL="jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" \
                            -e SPRING_DATASOURCE_USERNAME="${MYSQL_USERNAME}" \
                            -e SPRING_DATASOURCE_PASSWORD="${MYSQL_PASSWORD}" \
                            -e SERVER_PORT=9093 \
                            "$NEW_BACKEND_IMAGE"

                        echo ""
                        echo "Waiting for backend..."

                        BACKEND_READY=false

                        for i in $(seq 1 30)
                        do
                            echo "Backend health check ${i}/30..."

                            if curl -s \
                                --fail \
                                http://127.0.0.1:${BACKEND_PORT}/etmsys/v1/user/captcha \
                                > /tmp/backend-prod-response.txt 2>/dev/null
                            then
                                BACKEND_READY=true
                                echo "Backend is healthy."
                                break
                            fi

                            sleep 5
                        done

                        if [ "$BACKEND_READY" != "true" ]
                        then

                            echo ""
                            echo "========================================"
                            echo "BACKEND DEPLOYMENT FAILED"
                            echo "========================================"

                            docker logs --tail 200 backend || true

                            echo ""
                            echo "Rolling back backend..."

                            docker rm -f backend 2>/dev/null || true

                            docker run -d \
                                --name backend \
                                --restart always \
                                -p ${BACKEND_PORT}:9093 \
                                -e SPRING_DATASOURCE_URL="jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC" \
                                -e SPRING_DATASOURCE_USERNAME="${MYSQL_USERNAME}" \
                                -e SPRING_DATASOURCE_PASSWORD="${MYSQL_PASSWORD}" \
                                -e SERVER_PORT=9093 \
                                "$OLD_BACKEND_IMAGE"

                            echo "Backend rollback completed."

                            exit 1
                        fi

                        echo ""
                        echo "========================================"
                        echo "STARTING NEW FRONTEND"
                        echo "========================================"

                        docker run -d \
                            --name frontend \
                            --restart always \
                            -p ${FRONTEND_PORT}:80 \
                            "$NEW_FRONTEND_IMAGE"

                        echo ""
                        echo "Waiting for frontend..."

                        sleep 5

                        docker exec frontend nginx -t

                        curl --fail \
                            --show-error \
                            --retry 10 \
                            --retry-delay 2 \
                            --retry-connrefused \
                            http://127.0.0.1:${FRONTEND_PORT}/

                        echo ""
                        echo "========================================"
                        echo "FRONTEND DEPLOYMENT SUCCESS"
                        echo "========================================"

                        echo ""
                        echo "========================================"
                        echo "APPLICATION STATUS"
                        echo "========================================"

                        docker ps \
                            --filter name=backend \
                            --filter name=frontend

                        echo ""
                        echo "Backend:"
                        echo "http://172.16.1.89:${BACKEND_PORT}"

                        echo ""
                        echo "Frontend:"
                        echo "http://172.16.1.89:${FRONTEND_PORT}"

                        echo ""
                        echo "========================================"
                        echo "DEPLOYMENT SUCCESS"
                        echo "========================================"
                    '''
                }
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
                 CI/CD PIPLETED SUCCESSFULLY
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


            Backend Image:
            ${BACKEND_IMAGE}:${BUILD_NUMBER}


            Frontend Image:
            ${FRONTEND_IMAGE}:${BUILD_NUMBER}


            Check the failed stage in Jenkins Console Output.


            ==========================================
                 PIPELINE FAILED
            ==========================================
            """
        }


        always {

            sh '''
                echo ""
                echo "========================================"
                echo "CLEANUP"
                echo "========================================"

                echo ""
                echo "Cleaning temporary backend test container..."

                docker rm -f etmsys-backend-test 2>/dev/null || true


                echo ""
                echo "Cleaning temporary frontend test container..."

                docker rm -f etmsys-frontend-test 2>/dev/null || true


                echo ""
                echo "Logging out from Docker Hub..."

                docker logout 2>/dev/null || true


                echo ""
                echo "Cleanup completed."
            '''
        }
    }
}
