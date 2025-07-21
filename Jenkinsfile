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
                    sh "docker build -t test-image -f Dockerfile.test ."
                    sh "docker run test-image flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics"
                    //sh "docker run test-image pytest"
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo 'Building the production Docker image...'
                script {
                    withCredentials([aws(credentialsId: 'AWS_CREDS')]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY_URI}"
                
                        def dockerImage = docker.build("${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}")
                        echo "Image ${dockerImage.id} built."
                        dockerImage.push()                    

                        echo "Image pushed to ECR successfully."
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "Deploying to ${EC2_HOSTNAME}..."
                sshagent(credentials: ['EC2_SSH_KEY']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOSTNAME} << ENDSSH
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY_URI}
                        docker pull ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}

                        echo 'Stopping and removing the old container...'
                        docker stop mi-flask-app || true
                        docker rm mi-flask-app || true

                        echo 'Starting new container...'
                        docker run -d --name mi-flask-app -p 80:5000 ${ECR_REGISTRY_URI}/${ECR_REPOSITORY_NAME}:${IMAGE_TAG}
                        exit 0
                    """
                }
            }
        }
    }
}

