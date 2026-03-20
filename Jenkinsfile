pipeline {
    agent any

    environment {
        SOCKET_SECURITY_API_TOKEN = credentials("${params.SOCKET_API_CREDENTIAL_ID ?: 'socket-api-key'}")
        REPO_NAME = "${params.REPO_NAME ?: env.JOB_NAME}"
    }

    stages {
        stage('Socket Security Scan') {
            steps {
                sh """
                    socketcli \
                        --target-path . \
                        --repo ${REPO_NAME} \
                        --default-branch \
                        --reach \
                        --reach-ecosystems go \
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
