pipeline {
    agent any

    environment {
        SOCKET_SECURITY_API_TOKEN = credentials('socket-api-key')
        REPO_NAME = "${env.REPO_NAME ?: 'project-x'}"
    }

    stages {
        stage('Checkout Target Project') {
            steps {
                git url: env.TARGET_REPO_URL, branch: "${env.TARGET_BRANCH ?: 'master'}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install --production'
            }
        }

        stage('Socket Security Scan') {
            steps {
                sh """
                    socketcli \
                        --target-path . \
                        --repo ${env.REPO_NAME ?: 'project-x'} \
                        --default-branch \
                        --reach \
                        --reach-ecosystems npm \
                        --disable-blocking \
                        --integration api
                """
            }
        }
    }

    post {
        always {
            echo 'Scan complete. View results at https://socket.dev/dashboard'
        }
    }
}
