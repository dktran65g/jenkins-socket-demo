pipeline {
    agent any

    environment {
        SOCKET_SECURITY_API_TOKEN = credentials('socket-api-key')
        REPO_NAME = "${env.REPO_NAME ?: 'project-x'}"
    }

    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm install --ignore-scripts'
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
