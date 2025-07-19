pipeline {

    agent any

    environment {
        ECR_REGISTRY_URI = "291041007750.dkr.ecr.us-east-2.amazonaws.com"
        ECR_REPOSITORY_NAME = "mi-flask-app"
        AWS_REGION = "us-east-2"
        IMAGE_TAG = "${BUILD_NUMBER}"
        EC2_HOSTNAME = "ec2-18-118-173-125.us-east-2.compute.amazonaws.com"
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
                    //bat "docker run test-image pytest"
                }
            }
        }

        /*stage('Build & Push Docker Image') {
            steps {
                echo 'Building the production Docker image...'
                script {
                    def dockerImage = docker.build("${ECR_REPOSITORY_NAME}:${IMAGE_TAG}")

                    echo "Image ${dockerImage.id} built."
                    echo "Publishing image to AWS ECR: ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}"
                    bat "aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ${ECR_REGISTRY_URI}"
                    bat "docker tag mi-flask-app:${IMAGE_TAG} ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}"
                    bat "docker push ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo 'Deploying to EC2...'
                sshagent(credentials: ['EC2_SSH_KEY']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@18.118.173.125 <<'ENDSSH'
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY_URI}

                            docker pull ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}

                            docker stop mi-flask-app || true
                            docker rm mi-flask-app || true

                            docker run -d --name mi-flask-app -p 80:5000 ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}
                        ENDSSH
                    '''
                }
            }
        }*/
        stage('Deploy to EC2') {
            steps {
                echo 'Deploying to EC2 on Windows agent...'
                withCredentials([file(credentialsId: 'EC2_SSH_PEM', variable: 'PEM_FILE')]) { // Credentials are stored in Jenkins and we need to chmod 400 the file
                    bat """
                        icacls "%PEM_FILE%" /inheritance:r
                        icacls "%PEM_FILE%" /grant:r "NT AUTHORITY\\SYSTEM:R"
                        ssh -i %PEM_FILE% -o StrictHostKeyChecking=no ubuntu@${EC2_HOSTNAME} " ^
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY_URI} && ^
                            docker pull ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG} && ^
                            docker stop mi-flask-app || exit 0 && ^
                            docker rm mi-flask-app || exit 0 && ^
                            docker run -d --name mi-flask-app -p 80:5000 ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}
                        "
                    """
                }
            }
        }
    }
}
