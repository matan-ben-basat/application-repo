pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "992382545251"
        AWS_DEFAULT_REGION = "us-east-1"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        ECR_REPOSITORY = "ci-cd-exam-calculator-app"
        ECR_URL = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
        IMAGE_TAG = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
    }
    stages {
        stage('Build Container Image') {
            steps {
                dir('calculator-app-v2') {
                    sh "docker build -t ${ECR_URL}:${IMAGE_TAG} ."
                }
            }
        }
        stage('Test') {
            steps {
                sh "docker run --name test-runner ${ECR_URL}:${IMAGE_TAG} python -m unittest discover -s tests -v"
            }
            post {
                always {
                    echo "Tests run completed."
                }
                cleanup {
                    sh "docker rm test-runner || true"
                }
            }
        }
        stage('Push to ECR') {
            when { 
                branch 'PR-*' 
            }
            steps {
                // שימוש בקונטיינר זמני של AWS CLI המכיל את הכלים המובנים כדי לבצע לוגין ולדחוף ל-ECR בבטחה
                withDockerContainer(image: 'amazon/aws-cli') {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                }
                sh "docker push ${ECR_URL}:${IMAGE_TAG}"
            }
        }
    }
}
