pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        ACCOUNT_ID = '<aws_account_id>'
        IMAGE_REPO = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/nginx-app"
        APP_NAME = 'nginx-app'
        SONARQUBE_ENV = 'SonarQube'  // Jenkins SonarQube server name
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Static Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=$APP_NAME \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $IMAGE_REPO:$BRANCH_NAME .'
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    sh '''
                        echo "🔍 Scanning image for vulnerabilities using Trivy..."
                        trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE_REPO:$BRANCH_NAME || {
                            echo "❌ Critical vulnerabilities found. Failing build."
                            exit 1
                        }
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                        docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                script {
                    sh 'docker push $IMAGE_REPO:$BRANCH_NAME'
                }
            }
        }

        stage('Configure EKS Cluster Access') {
            steps {
                script {
                    def cluster_name = ""
                    if (env.BRANCH_NAME == 'dev') {
                        cluster_name = "eks-dev-cluster"
                    } else if (env.BRANCH_NAME == 'qa') {
                        cluster_name = "eks-qa-cluster"
                    } else if (env.BRANCH_NAME == 'stg') {
                        cluster_name = "eks-stg-cluster"
                    } else if (env.BRANCH_NAME == 'prod') {
                        cluster_name = "eks-prod-cluster"
                    }

                    sh "aws eks update-kubeconfig --name ${cluster_name} --region ${AWS_DEFAULT_REGION}"
                }
            }
        }

        stage('Helm Deploy to EKS') {
            steps {
                script {
                    sh '''
                        helm upgrade --install $APP_NAME ./helm/nginx-app \
                          --namespace $BRANCH_NAME \
                          --create-namespace \
                          --set image.repository=$IMAGE_REPO \
                          --set image.tag=$BRANCH_NAME
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "✅ Pipeline completed for branch: ${BRANCH_NAME}"
            cleanWs()
        }
    }
}
