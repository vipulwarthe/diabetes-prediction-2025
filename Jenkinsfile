pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        ACCOUNT_ID     = "717279727098"
        ECR_REPO_NAME  = "diabetes-streamlit-app"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        ECS_CLUSTER    = "diabetes-ecs-cluster"
    }

    stages {

        /* ✅ Checkout code */
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        /* ✅ Build Docker Image */
        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} .
                """
            }
        }

        /* ✅ Configure AWS CLI */
        stage('AWS Configure') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region ${AWS_REGION}
                    """
                }
            }
        }

        /* ✅ Authenticate Docker to ECR */
        stage('Login to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """
            }
        }

        /* ✅ Create ECR Repository (if not present) */
        stage('Create ECR Repo') {
            steps {
                sh """
                    echo "Checking if ECR repo exists..."

                    if ! aws ecr describe-repositories \
                        --repository-names ${ECR_REPO_NAME} \
                        --region ${AWS_REGION} 2>/dev/null; then

                        echo "Creating ECR repo..."
                        aws ecr create-repository \
                          --repository-name ${ECR_REPO_NAME} \
                          --image-scanning-configuration scanOnPush=true \
                          --region ${AWS_REGION}
                    else
                        echo "✅ ECR repo already exists!"
                    fi
                """
            }
        }

        /* ✅ Tag & Push Image to ECR */
        stage('Tag & Push Image') {
            steps {
                sh """
                    docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}

                    docker push \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                """
            }
        }

        /* ✅ Create ECS Cluster (Only once) */
        stage('Create ECS Cluster') {
            steps {
                sh """
                    CLUSTER_STATUS=\$(aws ecs describe-clusters \
                        --clusters ${ECS_CLUSTER} \
                        --query "clusters[0].status" \
                        --output text 2>/dev/null)

                    if [ "\$CLUSTER_STATUS" = "ACTIVE" ]; then
                        echo "✅ ECS Cluster already exists!"
                    else
                        echo "Creating ECS Cluster..."
                        aws ecs create-cluster --cluster-name ${ECS_CLUSTER} --region ${AWS_REGION}
                    fi
                """
            }
        }
    }

    post {
        success {
            echo "✅ Image pushed & ECS Cluster ready!"
            echo "👉 Now go to AWS console to create Task Definition & ECS Service manually."
        }
        failure {
            echo "❌ Pipeline Failed — Cleaning local Docker images..."
            sh """
                docker rmi -f ${ECR_REPO_NAME}:${IMAGE_TAG} || true
                docker image prune -f || true
            """
        }
    }
}
