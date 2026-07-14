pipeline {
    agent {
        docker {
            image 'docker:cli'
            // מיפוי ה-Socket של המארח לתוך הסוכן לצורך עבודה מאובטחת
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
   environment {
        // הכתובת המדויקת של ה-ECR שלך מהתמונה
        ECR_URL = "992382545251.dkr.ecr.us-east-1.amazonaws.com/ci-cd-exam-calculator-app"
        IMAGE_TAG = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
    }
    stages {
        // שלב 1: בניית האימג' מתוך תיקיית האפליקציה (חלק ג', שלב 1)
        stage('Build Container Image') {
            steps {
                // כניסה לתיקייה calculator-app-v2 בה נמצא ה-Dockerfile
                dir('calculator-app-v2') {
                    sh "docker build -t ${ECR_URL}:${IMAGE_TAG} ."
                }
            }
        }
        
        // שלב 2: הרצת הבדיקות המקוריות של הפרויקט (חלק ג', שלב 2)
        stage('Test') {
            steps {
                // הרצת הבדיקות מתוך האימג' שנבנה
                // הפקודה תכשיל את הצינור באופן אוטומטי במקרה של שגיאה בבדיקות
                sh "docker run --name test-runner ${ECR_URL}:${IMAGE_TAG} python -m unittest discover -s tests -v"
            }
            post {
                always {
                    echo "Tests run completed."
                }
                cleanup {
                    // ניקוי הקונטיינר הזמני
                    sh "docker rm test-runner || true"
                }
            }
        }
        
        // שלב 3: דחיפה ל-ECR רק במידה ומדובר ב-PR (חלק ג', שלב 3)
        stage('Push to ECR') {
            when { 
                branch 'PR-*' 
            }
            steps {
                // התחברות מאובטחת ללא מפתחות קשיחים בקוד (באמצעות ה-Instance Role של שרת ה-Jenkins)
                sh "aws ecr get-login-password --region your-region | docker login --username AWS --password-stdin ${ECR_URL}"
                sh "docker push ${ECR_URL}:${IMAGE_TAG}"
            }
        }
    }
}
