pipeline {
    agent {
        docker {
            // שימוש באימג' הרשמי של דוקר שמאפשר הרצת פקודות דוקר בפנים
            image 'docker:cli'
            // מיפוי הסוקט והרצת הסוכן כ-root למניעת בעיות הרשאות מול המארח
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }

    environment {
        AWS_ACCOUNT_ID = "992382545251"
        AWS_DEFAULT_REGION = "us-east-1"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        ECR_REPOSITORY = "ci-cd-exam-calculator-app"
        ECR_URL = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
    }

    stages {
        // ==================== שלבי ה-CI ====================
        
        stage('Initialize Environment') {
            steps {
                script {
                    // הגדרה חוקית של טאג האימג' בתוך בלוק סקריפט
                    if (env.CHANGE_ID) {
                        env.IMAGE_TAG = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                    } else {
                        def commitSha = env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'latest'
                        env.IMAGE_TAG = "candidate-${commitSha}"
                    }
                    echo "Target Image Tag will be: ${env.IMAGE_TAG}"
                }
                // התקנת הכלים החסרים באימג' (AWS CLI ו-SSH) בזמן אמת
                sh "apk add --no-cache aws-cli openssh-client curl"
            }
        }

        stage('Build Container Image') {
            steps {
                sh "docker build -t ${ECR_URL}:${env.IMAGE_TAG} ."
            }
        }

        stage('Test') {
            steps {
                sh "docker run --name test-runner ${ECR_URL}:${env.IMAGE_TAG} python -m unittest discover -s tests -v"
            }
            post {
                always {
                    echo "Tests run completed."
                    sh "docker rm -f test-runner || true"
                }
            }
        }

        // ==================== שלבי ה-CD ====================

        stage('Push PR Image to ECR') {
            when { 
                branch 'PR-*' 
            }
            steps {
                sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                sh "docker push ${ECR_URL}:${env.IMAGE_TAG}"
            }
        }

        stage('Push Master Image to ECR') {
            when { 
                branch 'main' 
            }
            steps {
                sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                sh "docker push ${ECR_URL}:${env.IMAGE_TAG}"
                
                // תיוג כ-latest עבור סביבת הפרודקשן
                sh "docker tag ${ECR_URL}:${env.IMAGE_TAG} ${ECR_URL}:latest"
                sh "docker push ${ECR_URL}:latest"
            }
        }

        stage('Deploy to Prod EC2') {
            when { 
                branch 'main' 
            }
            steps {
                echo "Deploying to Production Server..."
                withCredentials([sshUserPrivateKey(credentialsId: 'prod-ssh-key', keyFileVariable: 'IDENTITY')]) {
                    sh '''
                        ssh -i ${IDENTITY} -o StrictHostKeyChecking=no ubuntu@44.202.68.154 "
                            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 992382545251.dkr.ecr.us-east-1.amazonaws.com &&
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
            when { 
                branch 'main' 
            }
            steps {
                echo "Verifying application health..."
                retry(5) {
                    sleep 5
                    sh "curl -f http://44.202.68.154/health"
                }
            }
        }
    }
}
