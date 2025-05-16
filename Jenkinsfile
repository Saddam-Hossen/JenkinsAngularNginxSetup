pipeline {
    agent any

    environment {
        PROD_HOST  = credentials('DO_HOST')     // e.g., 159.89.172.251
        PROD_USER  = credentials('DO_USER')     // e.g., root
        DEPLOY_DIR = '/www/wwwroot/CITSNVN/jenkins/angular'
        BUILD_DIR  = 'dist/my-project'          // ✅ Corrected build directory
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Saddam-Hossen/JenkinsAngularProject.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }
        stage('Verify Files') {
            steps {
                bat 'dir src\\app\\environments'
            }
        }
        stage('Build Angular Project') {
            steps {
                bat 'ng build --configuration=production'
            }
        }
        


        stage('Deploy to DigitalOcean') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'DO_SSH_KEY', keyFileVariable: 'SSH_KEY')]) {
                    bat """
                    "C:\\Program Files\\Git\\bin\\bash.exe" -c \
                    "scp -o StrictHostKeyChecking=no -i \\"$SSH_KEY\\" -r ${BUILD_DIR}/* ${PROD_USER}@${PROD_HOST}:${DEPLOY_DIR}"
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Angular build and deploy complete!'
        }
        failure {
            echo '❌ Build or deployment failed. Check console output.'
        }
    }
}
