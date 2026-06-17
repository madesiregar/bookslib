pipeline {
    agent any
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('SAST - Security Scan') {
            parallel {
                stage('Bandit (Python)') {
                    steps {
                        sh 'docker run --rm -v ${WORKSPACE}/reviews-service:/app cytopia/bandit -r /app -f json -o /app/bandit-report.json || true'
                    }
                }
                stage('Gosec (Go)') {
                    steps {
                        sh 'docker run --rm -v ${WORKSPACE}/auth-service:/app -w /app securego/gosec:latest -fmt=json -out=/app/gosec-report.json ./... || true'
                    }
                }
            }
        }
        stage('Build & Test') {
            steps {
                sh 'docker compose build'
            }
        }
        stage('Image Scan - Trivy') {
            steps {
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}:/workspace aquasec/trivy image --severity HIGH,CRITICAL --format json -o /workspace/trivy-report.json bookslib-auth-service || true'
            }
        }
    }
    
    // Tambahkan bagian ini di sini:
    post {
        always {
            archiveArtifacts artifacts: '**/bandit-report.json, **/gosec-report.json, **/trivy-report.json', allowEmptyArchive: true
        }
    }
}