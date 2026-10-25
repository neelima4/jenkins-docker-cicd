pipeline {

    agent any

    environment {
        DOCKER_IMAGE = 'neelima4/jenkins-docker-cicd'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t $DOCKER_IMAGE:${BUILD_NUMBER} \
                      -t $DOCKER_IMAGE:latest \
                      .
                '''
            }
        }

        stage('Login and Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                          -u "$DOCKER_USER" \
                          --password-stdin

                        docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                        docker push $DOCKER_IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy New Version') {
            steps {
                sh '''
                    echo "Deploying build ${BUILD_NUMBER}"

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose pull app

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose down

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose up -d
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking application health..."

                    for i in $(seq 1 12); do

                        if curl --fail --silent \
                            http://localhost:8081/actuator/health \
                            | grep -q '"status":"UP"'; then

                            echo "Application is healthy"
                            echo "Build ${BUILD_NUMBER} deployed successfully"

                            APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                                docker compose ps

                            exit 0
                        fi

                        echo "Application not ready. Attempt $i/12"
                        sleep 5
                    done

                    echo "Deployment failed health check"

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose ps || true

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose logs app || true

                    exit 1
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    docker image prune -f
                '''
            }
        }
    }

    post {

        success {
            echo "======================================"
            echo "PIPELINE SUCCESS"
            echo "Build ${BUILD_NUMBER} is running"
            echo "======================================"
        }

        failure {

            script {

                int previousBuild = env.BUILD_NUMBER.toInteger() - 1

                echo "======================================"
                echo "DEPLOYMENT FAILED"
                echo "Failed build: ${BUILD_NUMBER}"
                echo "Rollback target: ${previousBuild}"
                echo "======================================"

                if (previousBuild <= 0) {
                    echo "No previous build available for rollback"
                    return
                }

                /*
                 * Step 1:
                 * Check whether previous Docker image exists.
                 */
                int pullResult = sh(
                    script: """
                        echo "Pulling previous image..."
                        docker pull ${DOCKER_IMAGE}:${previousBuild}
                    """,
                    returnStatus: true
                )

                if (pullResult != 0) {
                    echo "======================================"
                    echo "ROLLBACK CANNOT START"
                    echo "Image ${DOCKER_IMAGE}:${previousBuild} was not found"
                    echo "======================================"
                    return
                }

                echo "Previous Docker image found"

                /*
                 * Step 2:
                 * Stop failed deployment.
                 */
                sh(
                    script: """
                        echo "Stopping failed deployment..."

                        APP_IMAGE=${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            docker compose down
                    """,
                    returnStatus: true
                )

                /*
                 * Step 3:
                 * Deploy previous Docker image.
                 */
                int rollbackStart = sh(
                    script: """
                        echo "Starting rollback build ${previousBuild}..."

                        APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                            docker compose up -d
                    """,
                    returnStatus: true
                )

                if (rollbackStart != 0) {

                    echo "======================================"
                    echo "ROLLBACK CONTAINER START FAILED"
                    echo "======================================"

                    sh(
                        script: """
                            APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                                docker compose ps
                        """,
                        returnStatus: true
                    )

                    sh(
                        script: """
                            APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                                docker compose logs app
                        """,
                        returnStatus: true
                    )

                    return
                }

                /*
                 * Step 4:
                 * Verify rollback health.
                 */
                echo "Rollback containers started"
                echo "Waiting for rollback application..."

                int healthResult = sh(
                    script: """
                        for i in \$(seq 1 12); do

                            if curl --fail --silent \
                                http://localhost:8081/actuator/health \
                                | grep -q '"status":"UP"'; then

                                echo "Rollback application is healthy"
                                exit 0
                            fi

                            echo "Rollback health check attempt \$i/12"
                            sleep 5
                        done

                        exit 1
                    """,
                    returnStatus: true
                )

                /*
                 * Step 5:
                 * Report rollback result.
                 */
                if (healthResult == 0) {

                    echo "======================================"
                    echo "ROLLBACK SUCCESSFUL"
                    echo "Running build: ${previousBuild}"
                    echo "======================================"

                    sh(
                        script: """
                            APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                                docker compose ps
                        """,
                        returnStatus: true
                    )

                    sh(
                        script: """
                            docker inspect \
                                jenkins-docker-cicd-app-1 \
                                --format='{{.Config.Image}}'
                        """,
                        returnStatus: true
                    )

                } else {

                    echo "======================================"
                    echo "ROLLBACK FAILED"
                    echo "Build ${previousBuild} is not healthy"
                    echo "======================================"

                    sh(
                        script: """
                            APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                                docker compose ps
                        """,
                        returnStatus: true
                    )

                    sh(
                        script: """
                            APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                                docker compose logs app
                        """,
                        returnStatus: true
                    )
                }
            }
        }
    }
}
