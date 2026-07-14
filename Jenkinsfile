pipeline {
    agent {
        docker {
            image 'docker:cli'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        ECR_URL = "992382545251.dkr.ecr.us-east-1.amazonaws.com/ci-cd-exam-calculator-app"
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
                sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_URL}"
                sh "docker push ${ECR_URL}:${IMAGE_TAG}"
            }
        }
    }
}
