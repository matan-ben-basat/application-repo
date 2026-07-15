pipeline {
    agent {
        docker {
            image 'docker:cli'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        ECR_URL = "992382545251.dkr.ecr.us-east-1.amazonaws.com/ci-cd-exam-calculator-app"
        IMAGE_TAG = env.CHANGE_ID ? "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}" : "candidate-${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'latest'}"
    }
    stages {
        stage('Initialize Environment') {
            steps {
                script {
                    echo "Determined Target Image Tag: ${IMAGE_TAG}"
                }
                sh "apk add --no-cache aws-cli openssh-client curl"
            }
        }
        stage('Build Container Image') {
            steps {
                dir('calculator-app-v2') {
                    sh "docker build -t ${ECR_URL}:${IMAGE_TAG} ."
                }
            }
        }
        stage('Test') {
            steps {
                sh "docker run --rm ${ECR_URL}:${IMAGE_TAG} python -m unittest discover -s tests -v"
            }
        }
        stage('Push PR Image to ECR') {
            when { branch 'PR-*' }
            steps {
                sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_URL}"
                sh "docker push ${ECR_URL}:${IMAGE_TAG}"
            }
        }
        stage('Push Master Image to ECR') {
            when { branch 'main' }
            steps {
                sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_URL}"
                sh "docker push ${ECR_URL}:${IMAGE_TAG}"
                sh "docker tag ${ECR_URL}:${IMAGE_TAG} ${ECR_URL}:latest"
                sh "docker push ${ECR_URL}:latest"
            }
        }
        stage('Deploy to Prod EC2') {
            when { branch 'main' }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'prod-ssh-key', keyFileVariable: 'IDENTITY')]) {
                    sh '''
                        ssh -i ${IDENTITY} -o StrictHostKeyChecking=no ubuntu@44.202.68.154 "
                            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 992382545251.dkr.ecr.us-east-1.amazonaws.com/ci-cd-exam-calculator-app &&
                            docker stop app-container || true &&
                            docker rm app-container || true &&
                            docker pull 992382545251.dkr.ecr.us-east-1.amazonaws.com/ci-cd-exam-calculator-app:latest &&
                            docker run -d --name app-container -p 80:5000 992382545251.dkr.ecr.us-east-1.amazonaws.com/ci-cd-exam-calculator-app:latest
                        "
                    '''
                }
            }
        }
        stage('Health Verification') {
            when { branch 'main' }
            steps {
                retry(5) {
                    sleep 5
                    sh "curl -f http://44.202.68.154/health"
                }
            }
        }
    }
}