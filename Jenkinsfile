pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "hanif040/kfc-static"
        DOCKER_TAG = "latest"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
            }
        }

        stage('Verify Image') {
            steps {
                sh "docker run -d -p 8081:80 --name kfc-verify ${DOCKER_IMAGE}:${DOCKER_TAG}"
                sh "sleep 3 && curl -f http://localhost:8081 || exit 1"
                sh "docker stop kfc-verify && docker rm kfc-verify"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded — image pushed to Docker Hub."
        }
        failure {
            echo "Pipeline failed — check logs above."
        }
        always {
            sh "docker logout"
        }
    }
}
