pipeline {

    agent any

    environment {
        ECR_REGISTRY_URI = "291041007750.dkr.ecr.us-east-2.amazonaws.com/mi-flask-app"
        ECR_REPOSITORY_NAME = "mi-flask-app"
        AWS_REGION = "us-east-2"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Clon repository...'
                checkout scm
            }
        }

        stage('Lint & Test') {
            steps {
                echo 'Analyzing and testing the code...'
                script {
                    bat "docker build -t test-image -f Dockerfile.test ."
                    bat "docker run test-image flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics"
                    bat "docker run test-image pytest"
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo 'Building the production Docker image...'
                script {
                    def dockerImage = docker.build("${ECR_REPOSITORY_NAME}:${IMAGE_TAG}")

                    echo "Image ${dockerImage.id} built."
                    echo "Publishing image to AWS ECR: ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}"
                    docker.withRegistry( 'https://' + ECR_REGISTRY_URI, 'AWS_CREDS' ) {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo 'Deploying to EC2...'
                sshagent(credentials: ['EC2_SSH_KEY']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@3.138.186.60 <<'ENDSSH'
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY_URI}

                            docker pull ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}

                            docker stop mi-flask-app || true
                            docker rm mi-flask-app || true

                            docker run -d --name mi-flask-app -p 80:5000 ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}
                        ENDSSH
                    '''
                }
            }
        }
    }
}
