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
                    echo "Deploying version ${BUILD_NUMBER}"

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose pull app

                    docker compose down

                    APP_IMAGE=$DOCKER_IMAGE:${BUILD_NUMBER} \
                        docker compose up -d
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Waiting for application..."

                    for i in $(seq 1 12); do

                        if curl --fail --silent \
                            http://localhost:8081/actuator/health \
                            | grep -q '"status":"UP"'; then

                            echo "Application is healthy"
                            curl http://localhost:8081/actuator/health
                            exit 0
                        fi

                        echo "Application not ready. Attempt $i/12"
                        sleep 5
                    done

                    echo "Deployment failed health check"

                    docker compose ps
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
            echo "Build ${BUILD_NUMBER} deployed successfully"
        }

        failure {

            script {

                int previousBuild =
                    env.BUILD_NUMBER.toInteger() - 1

                echo "Deployment failed"
                echo "Rolling back to build ${previousBuild}"

                if (previousBuild > 0) {

                    sh """
                        echo "Pulling previous image..."

                        docker pull \
                            ${DOCKER_IMAGE}:${previousBuild}

                        docker compose down

                        echo "Deploying previous version..."

                        APP_IMAGE=${DOCKER_IMAGE}:${previousBuild} \
                            docker compose up -d

                        echo "Waiting for rollback deployment..."

                        sleep 10

                        docker compose ps
                    """

                } else {

                    echo "No previous build available for rollback"
                }
            }
        }
    }
}
