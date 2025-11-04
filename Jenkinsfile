pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        ACCOUNT_ID     = "717279727098"
        ECR_REPO_NAME  = "diabetes-streamlit-app"
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

        stage('Run OWASP Dependency Check') {
            steps {
                sh '''
                    echo "Running OWASP Dependency Check..."
                    dependency-check.sh \
                        --project "diabetes-app" \
                        --scan . \
                        --format HTML \
                        --out dependency-check-report
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/**/*', fingerprint: true
                }
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
                        aws configure set default.region us-east-1
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
            echo "✅ Successfully deployed with OWASP & Trivy Security Scans!"
        }
        failure {
            echo "❌ Pipeline Failed — Check Security Reports!"
        }
    }
}





