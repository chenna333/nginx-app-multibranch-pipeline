pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        IMAGE_REPO = '<aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/nginx-app'
        APP_NAME = 'nginx-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $IMAGE_REPO:$BRANCH_NAME .'
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $IMAGE_REPO
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                script {
                    sh '''
                        docker push $IMAGE_REPO:$BRANCH_NAME
                    '''
                }
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                script {
                    sh '''
                        helm upgrade --install $APP_NAME ./helm/nginx-app \
                          --set image.repository=$IMAGE_REPO \
                          --set image.tag=$BRANCH_NAME
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed.'
        }
    }
}
