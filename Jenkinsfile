pipeline {
    agent any

    environment {
        PROJECT_NAME = "wanderlust"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Getting latest code from GitHub...'
                checkout scm
            }
        }

        stage('Check Environment') {
            steps {
                sh '''
                    echo "Checking Docker..."
                    docker --version

                    echo "Checking Docker Compose..."
                    docker compose version
                '''
            }
        }

        stage('Build Application') {
            steps {
                sh '''
                    echo "Building Docker images..."
                    docker compose build
                '''
            }
        }

        stage('Stop Old Containers') {
            steps {
                sh '''
                    echo "Stopping old containers..."
                    docker compose down || true
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    echo "Starting Wanderlust application..."
                    docker compose up -d
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking running containers..."
                    docker compose ps

                    echo "All Docker containers:"
                    docker ps
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
WANDERLUST DEPLOYMENT SUCCESSFUL
========================================
'''
        }

        failure {
            echo '''
========================================
WANDERLUST DEPLOYMENT FAILED
========================================
'''
        }
    }
}
