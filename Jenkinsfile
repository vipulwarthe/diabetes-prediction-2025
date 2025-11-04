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

        stage('Install OWASP Dependency Check') {
            steps {
                sh """
                    echo 'Installing OWASP Dependency Check...'
                    wget https://github.com/jeremylong/DependencyCheck/releases/download/v10.0.3/dependency-check-10.0.3-release.zip -O dc.zip
                    unzip dc.zip
                    chmod +x dependency-check/bin/dependency-check.sh
                """
            }
        }

        stage('Run OWASP Dependency Check') {
            steps {
                sh """
                    echo 'Running OWASP Dependency Check...'
                    dependency-check/bin/dependency-check.sh \
                        --project "diabetes-app" \
                        --scan . \
                        --format HTML \
                        --out dependency-check-report
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/*', fingerprint: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    echo 'Installing Trivy Scanner...'
                    apt-get update || true
                    apt-get install wget -y || true
                    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                    echo deb https://aquasecurity.github.io/trivy-repo/deb generic main | sudo tee /etc/apt/sources.list.d/trivy.list
                    apt-get update || true
                    apt-get install trivy -y || true

                    echo 'Running Trivy Vulnerability Scan...'
                    trivy image --exit-code 0 --format table --severity HIGH,CRITICAL ${ECR_REPO_NAME}:${IMAGE_TAG} > trivy-report.txt
                """
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
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """
            }
        }

        stage('Tag & Push Image to ECR') {
            steps {
                sh """
                    docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                    docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy New Image to ECS') {
            steps {
                sh """
                    aws ecs update-service \
                        --cluster ${ECS_CLUSTER} \
                        --service ${ECS_SERVICE} \
                        --force-new-deployment \
                        --region ${AWS_REGION}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployment Successful — Streamlit App Updated!"
        }
        failure {
            echo "❌ Pipeline Failed!"
        }
    }
}

