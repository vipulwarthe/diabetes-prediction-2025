pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        ACCOUNT_ID     = "717279727098"
        ECR_REPO_NAME  = "diabetes-streamlit-app"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        ECS_CLUSTER    = "diabetes-ecs-cluster"
        TASK_FAMILY    = "diabetes-task-def"
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

        stage('Create ECR Repo') {
            steps {
                sh '''
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

        /* ✅ CREATE ECS CLUSTER (NO SERVICE) */
        stage('Create ECS Cluster') {
            steps {
                sh '''
                    if ! aws ecs describe-clusters --clusters ${ECS_CLUSTER} \
                        --region ${AWS_REGION} | grep "ACTIVE" >/dev/null; then

                        echo "Creating ECS Cluster..."
                        aws ecs create-cluster --cluster-name ${ECS_CLUSTER}

                    else
                        echo "✅ ECS Cluster already exists!"
                    fi
                '''
            }
        }

        /* ✅ REGISTER TASK DEFINITION ONLY */
        stage('Create Task Definition') {
            steps {
                sh '''
                    echo "Generating task definition JSON..."

                    cat <<EOF > taskdef.json
{
  "family": "${TASK_FAMILY}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "diabetes-container",
      "image": "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "essential": true
    }
  ]
}
EOF

                    echo "Registering task definition..."
                    aws ecs register-task-definition \
                        --cli-input-json file://taskdef.json
                '''
            }
        }

        /* ✅ No ECS Service stage (removed as requested) */
    }

    post {
        success {
            echo "✅ ECS Cluster & Task Definition created successfully!"
            echo "👉 Now go to ECS console → choose the cluster → Run Task manually."
        }

        failure {
            echo "❌ Pipeline Failed — cleaning images..."
            sh '''
                docker rmi -f ${ECR_REPO_NAME}:${IMAGE_TAG} || true
                docker image prune -f || true
            '''
        }
    }
}

