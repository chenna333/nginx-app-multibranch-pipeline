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
                    echo "🔨 Building Docker image..."
                    sh 'docker build -t $IMAGE_REPO:$BRANCH_NAME .'
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    echo "🔍 Scanning image for vulnerabilities using Trivy..."
                    // Install Trivy if not already installed
                    sh '''
                        if ! command -v trivy >/dev/null 2>&1; then
                            echo "Installing Trivy..."
                            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
                            sudo mv trivy /usr/local/bin/
                        fi

                        echo "Running Trivy scan on image..."
                        trivy image --exit-code 0 --severity LOW,MEDIUM --no-progress $IMAGE_REPO:$BRANCH_NAME
                        trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress $IMAGE_REPO:$BRANCH_NAME || {
                            echo "❌ High or Critical vulnerabilities found! Failing build."
                            exit 1
                        }
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    echo "🔑 Logging in to AWS ECR..."
                    sh '''
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $IMAGE_REPO
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                script {
                    echo "🚀 Pushing image to ECR..."
                    sh '''
                        docker push $IMAGE_REPO:$BRANCH_NAME
                    '''
                }
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                script {
                    echo "⚙️ Deploying to EKS using Helm..."
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
            echo '✅ Pipeline execution completed.'
        }
        failure {
            echo '❌ Pipeline failed. Check above logs for details.'
        }
    }
}
