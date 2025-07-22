pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Starting build for container: my-container'
                // REPLACE the next line with your real build command(s) for Windows
                bat 'echo Building the project...'
            }
        }

        stage('Test') {
            steps {
                // REPLACE the next line with your real test command(s) for Windows
                bat 'echo Running tests...'
            }
        }
    }

    post {
        always {
            // Clean up steps, if any
            bat 'echo Always clean up workspace or resources here...'
        }
        failure {
            echo 'Build failed. Container: my-container'
        }
    }
}
