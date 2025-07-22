pipeline {
    agent any

    environment {
        DOCKER_HOST = "npipe:////./pipe/docker_engine" // Windows Docker endpoint
        DOCKER_IMAGE = "sadie/my-web-app"
        DOCKER_TAG = "${env.BUILD_ID ?: 'latest'}"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = credentials('google-chat-webhook')
        DEPLOYMENT_URL = "http://localhost:8085"
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        HOST_PORT = "8085"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sadie19/jenkins.git',
                    credentialsId: ''
            }
        }
        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        bat """
                            docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        bat """
                            docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
                            docker build -t %DOCKER_IMAGE%:%DOCKER_TAG% .
                        """
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    bat """
                        docker push %DOCKER_IMAGE%:%DOCKER_TAG%
                        docker tag %DOCKER_IMAGE%:%DOCKER_TAG% %DOCKER_IMAGE%:latest
                        docker push %DOCKER_IMAGE%:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Convert the logic from sh to bat for Windows
                    bat """
                        REM Try to find any container using our HOST_PORT
                        FOR /F "tokens=*" %%i IN ('docker ps --filter "publish=%HOST_PORT%" --format "{{.Names}}"') DO (
                            SET CONTAINER_USING_PORT=%%i
                        )

                        REM If a container is using the port, stop and remove it
                        IF NOT "%CONTAINER_USING_PORT%"=="" (
                            echo Found container using port %HOST_PORT%: %CONTAINER_USING_PORT%
                            docker stop %CONTAINER_USING_PORT% || exit 0
                            docker rm %CONTAINER_USING_PORT% || exit 0
                        )

                        REM Remove our named container if it exists
                        docker inspect %CONTAINER_NAME% >nul 2>&1
                        IF %ERRORLEVEL%==0 (
                            docker stop %CONTAINER_NAME% || exit 0
                            docker rm %CONTAINER_NAME% || exit 0
                        )

                        REM Run the new container
                        docker run -d --name %CONTAINER_NAME% -p %HOST_PORT%:80 %DOCKER_IMAGE%:%DOCKER_TAG%
                    """
                }
            }
        }
    }

    post {
        always {
            node('built-in') {
                script {
                    bat """
                        docker logout || exit 0
                        docker ps -a --filter "name=%CONTAINER_NAME%" --format "{{.ID}}" > temp.txt
                        IF EXIST temp.txt (
                            FOR /F %%i IN (temp.txt) DO docker rm -f %%i || exit 0
                            del temp.txt || exit 0
                        )
                    """
                }
            }
        }
        success {
            node('built-in') {
                script {
                    def message = """
                    🚀 *Deployment Successful*
                    *Build*: #${env.BUILD_NUMBER}
                    *Image*: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                    *Container*: ${env.CONTAINER_NAME}
                    *URL*: ${env.DEPLOYMENT_URL}
                    """
                    sendGoogleChatNotification(message)
                }
            }
        }
        failure {
            node('built-in') {
                script {
                    def logs = bat(
                        script: """
                            docker inspect %CONTAINER_NAME% >nul 2>&1
                            IF %ERRORLEVEL%==0 (
                                docker logs --tail 50 %CONTAINER_NAME%
                            ) ELSE (
                                echo Container %CONTAINER_NAME% does not exist
                            )
                        """,
                        returnStdout: true
                    ).trim()

                    def message = """
                    🔴 *Deployment Failed*
                    *Build*: #${env.BUILD_NUMBER}
                    *Error*: ${currentBuild.currentResult}
                    *Logs*: ${logs}
                    """
                    sendGoogleChatNotification(message)
                }
            }
        }
    }
}

def sendGoogleChatNotification(String message) {
    node('built-in') {
        // Escape message for JSON and batch
        def escapedMessage = message.replace('"', '""').replace('\n', '^n').replace('%', '%%').replace('&', '^&')
        def payload = """{"text":"${escapedMessage}"}"""

        bat """
            curl -X POST ^
            -H "Content-Type: application/json" ^
            -d "${payload}" ^
            "%GOOGLE_CHAT_WEBHOOK%" || echo Notification failed
        """
    }
}
