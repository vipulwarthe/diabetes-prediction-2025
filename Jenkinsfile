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
                sh '''
                    sudo apt-get update -y
                    sudo apt-get install -y software-properties-common apt-transport-https

                    sudo add-apt-repository ppa:jeremylong/owasp-dependency-check -y
                    sudo apt-get update -y

                    sudo apt-get install -y dependency-check
                    dependency-check --version
                '''
            }
        }

        stage('Run Dependency Check') {
            steps {
                sh '''
                    dependency-check \
                        --project "diabetes-app" \
                        --scan . \
                        --format HTML \
                        --out dependency-check-report
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/*', fingerprint: true
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

        stage('Install Trivy') {
            steps {
                sh '''
                    sudo apt-get update -y
                    sudo apt-get install -y wget gnupg lsb-release

                    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                    echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main \
                        | sudo tee /etc/apt/sources.list.d/trivy.list

                    sudo apt-get update -y
                    sudo apt-get install -y trivy
                    trivy --version
                '''
            }
        }

        stage('Trivy Scan Image') {
            steps {
                sh '''
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
            echo "✅ Deployment Successful!"
        }
        failure {
            echo "❌ Deployment Failed!"
        }
    }
}



