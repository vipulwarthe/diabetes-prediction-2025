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

        /* -------------------------------
           OWASP DEPENDENCY CHECK (Docker)
           ------------------------------- */
        stage('Run OWASP Dependency Check') {
            agent {
                docker {
                    image 'owasp/dependency-check:latest'
                    args '-u 0:0 -v $WORKSPACE:/src -v dependency-check-data:/usr/share/dependency-check/data'
                }
            }
            steps {
                sh '''
                    echo "Running OWASP Dependency Check..."
                    dependency-check.sh \
                        --project "diabetes-app" \
                        --scan /src \
                        --format HTML \
                        --out /src/dependency-check-report
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/**/*', fingerprint: true
                }
            }
        }

        /* -------------------------------
           DOCKER IMAGE BUILD
           ------------------------------- */
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        /* -------------------------------
           TRIVY SCAN (Docker)
           ------------------------------- */
        stage('Trivy Scan Image') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args '--network host -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
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

        /* -------------------------------
           AWS CONFIGURE
           ------------------------------- */
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

        /* -------------------------------
           LOGIN TO ECR
           ------------------------------- */
        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin \
                      ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        /* -------------------------------
           TAG & PUSH
           ------------------------------- */
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

        /* -------------------------------
           DEPLOY TO ECS
           ------------------------------- */
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
            echo "✅ Deployment Successful with Secure DevSecOps pipeline!"
        }
        failure {
            echo "❌ Pipeline Failed — Check OWASP/Trivy reports."
        }
    }
}




