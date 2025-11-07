pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        ACCOUNT_ID     = "717279727098"
        ECR_REPO_NAME  = "diabetes-image-repo"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        ECS_CLUSTER    = "diabetes-ecs-cluster"
        ECS_SERVICE    = "diabetes-ecs-service"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Trivy Scan Image') {
            steps {
                sh '''
                    echo "Scanning Docker image with Trivy..."
                    trivy image --exit-code 0 \
                        --format table \
                        --severity HIGH,CRITICAL \
                        ${ECR_REPO_NAME}:${IMAGE_TAG} > trivy-report.txt
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
                }
            }
        }

        stage('AWS Configure') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region ${AWS_REGION}
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Create ECR Repo If Not Exists') {
            steps {
                sh '''
                    echo "Checking if ECR repo exists..."

                    if ! aws ecr describe-repositories \
                        --repository-names ${ECR_REPO_NAME} \
                        --region ${AWS_REGION} 2>/dev/null; then

                        echo "ECR repo not found. Creating..."
                        aws ecr create-repository \
                            --repository-name ${ECR_REPO_NAME} \
                            --image-scanning-configuration scanOnPush=true \
                            --region ${AWS_REGION}

                    else
                        echo "✅ ECR repo already exists!"
                    fi
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}

                    docker push \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                    aws ecs update-service \
                        --cluster ${ECS_CLUSTER} \
                        --service ${ECS_SERVICE} \
                        --force-new-deployment \
                        --region ${AWS_REGION}
                '''
            }
        }

    } // end stages

    post {
        success {
            echo "✅ Successfully deployed!"
        }

        failure {
            echo "❌ Pipeline Failed — Cleaning Docker Images..."

            sh '''
                echo "🧹 Removing failed local Docker image..."
                docker rmi -f ${ECR_REPO_NAME}:${IMAGE_TAG} || true

                echo "🧹 Removing dangling images..."
                docker image prune -f || true

                echo "🧹 Removing unused Docker layers..."
                docker system prune -f || true
            '''
        }
    }
}

