pipeline {
    agent any

    environment {
        COMPOSE_FILE = 'docker-compose.yaml'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                checkout scm
            }
        }

        stage('SAST - Bandit (Python)') {
            steps {
                echo 'Running Bandit on reviews-service...'
                sh '''
                    docker run --rm -v $(pwd)/reviews-service:/app \
                        cytopia/bandit bandit -r /app -f txt || true
                '''
            }
        }

        stage('SAST - Gosec (Go)') {
            steps {
                echo 'Running Gosec on auth-service...'
                sh '''
                    docker run --rm -v $(pwd)/auth-service:/app \
                        -w /app \
                        golang:1.20-alpine sh -c \
                        "go install github.com/securego/gosec/v2/cmd/gosec@latest && gosec ./..." || true
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building all services...'
                sh 'docker compose build'
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                echo 'Scanning images with Trivy...'
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        bookslib-auth-service || true

                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        bookslib-reviews-service || true
                '''
            }
        }

stage('Deploy') {
    steps {
        echo 'Deploying with docker compose...'
        withCredentials([file(credentialsId: 'bookslib-env', variable: 'ENV_FILE')]) {
            sh '''
                cp $ENV_FILE .env
                docker compose down || true
                docker compose up -d
            '''
        }
    }
}
