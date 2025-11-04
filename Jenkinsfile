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

        stage('Install OWASP Dependency Check (APT)') {
            steps {
                sh '''
                    echo "Installing OWASP Dependency Check via APT..."

                    sudo apt-get update -y
                    sudo apt-get install -y software-properties-common apt-transport-https

                    sudo add-apt-repository ppa:jeremylong/owasp-dependency-check -y
                    sudo apt-get update -y

                    sudo apt-get install -y dependency-check

                    dependency-check --version
                '''
            }
        }

        stage('Run OWASP Dependency Check') {
            steps {
                sh '''
                    echo "Running OWASP Dependency Check..."

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
                    echo "Installing Trivy Vulnerability Scanner..."

                    sudo apt-get update -y
                    sudo apt-get install -y wget gnupg lsb-release

                    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                    echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list

                    sudo apt-get update -y
                    sudo apt-get install -y trivy

                    trivy --version
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "Running Trivy scan on Docker image..."

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



