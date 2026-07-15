pipeline {
    agent {
        docker {
            // משתמשים באימג' שמכיל גם docker cli וגם aws cli מובנים כדי לעמוד באילוץ הסוכן
            image 'amazon/aws-cli:latest'
            // מיפוי ה-Socket בצורה מאובטחת כפי שנדרש באילוצים!
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        AWS_ACCOUNT_ID = "992382545251"
        AWS_DEFAULT_REGION = "us-east-1"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        ECR_REPOSITORY = "ci-cd-exam-calculator-app"
        ECR_URL = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
        IMAGE_TAG = env.CHANGE_ID ? "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}" : "candidate-${env.GIT_COMMIT.take(7)}"
    }
    stages {
        stage('Build Container Image') {
            steps {
                dir('calculator-app-v2') {
                    // פקודה זו תרוץ כעת בצורה חוקית מתוך סוכן ה-Docker!
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
                // מאחר שהסוכן מבוסס על amazon/aws-cli, פקודת ה-aws זמינה ישירות ללא הרצה של קונטיינר נוסף!
                sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                sh "docker push ${ECR_URL}:${IMAGE_TAG}"
            }
        }
    }
}
