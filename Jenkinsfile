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

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                    docker compose pull
                    docker compose down
                    docker compose up -d
                '''
            }
        }

        stage('Verify Deployment') {
 	 steps {
        	sh '''
            echo "Waiting for application to start..."

            for i in $(seq 1 12); do
                if curl --fail --silent http://localhost:8081 > /dev/null; then
                    echo "Application is healthy"
                    curl http://localhost:8081
                    exit 0
                fi

                echo "Application not ready yet. Attempt $i/12"
                sleep 5
            done

            echo "Application failed to become ready"
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
            echo 'CI/CD pipeline completed successfully'
        }

        failure {
            echo 'CI/CD pipeline failed'
        }
    }
}
