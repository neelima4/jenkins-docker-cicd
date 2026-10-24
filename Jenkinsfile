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
                    echo "Waiting for application health check..."

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
                        docker compose ps

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose logs app

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
            echo "Pipeline SUCCESS - Build ${BUILD_NUMBER} is running"
        }

        failure {
            script {

                int previousBuild = env.BUILD_NUMBER.toInteger() - 1

                echo "Pipeline failed"
                echo "Attempting rollback to build ${previousBuild}"

                if (previousBuild > 0) {

                    sh """
                        echo "---------------------------------------"
                        echo "ROLLBACK STARTED"
                        echo "Current failed build: ${BUILD_NUMBER}"
                        echo "Rollback target: ${previousBuild}"
                        echo "---------------------------------------"

                        echo "Pulling previous Docker image..."

                        docker pull \
                            ${DOCKER_IMAGE}:${previousBuild}

                        echo "Stopping failed deployment..."

                        APP_IMAGE=${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            docker compose down

                        echo "Starting previous working build..."

                        APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                            docker compose up -d

                        echo "Waiting for rollback application..."

                        for i in \$(seq 1 12); do

                            if curl --fail --silent \
                                http://localhost:8081/actuator/health \
                                | grep -q '"status":"UP"'; then

                                echo "---------------------------------------"
                                echo "ROLLBACK SUCCESSFUL"
                                echo "Running build: ${previousBuild}"
                                echo "---------------------------------------"

                                APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                                    docker compose ps

                                exit 0
                            fi

                            echo "Rollback application not ready. Attempt \$i/12"
                            sleep 5
                        done

                        echo "---------------------------------------"
                        echo "ROLLBACK FAILED"
                        echo "---------------------------------------"

                        APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                            docker compose ps

                        APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                            docker compose logs app

                        exit 1
                    """

                } else {
                    echo "No previous build is available for rollback"
                }
            }
        }
    }
}
