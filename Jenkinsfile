pipeline {
    agent any

    environment {
        COMPOSE_FILE = 'docker-compose.yaml'
        GITHUB_REPO = 'madesiregar/bookslib'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SAST - Bandit (Python)') {
            steps {
                sh '''
                    docker run --rm -v $(pwd)/reviews-service:/app cytopia/bandit -r /app -f json -o /app/bandit-report.json || true
                    cp reviews-service/bandit-report.json bandit-report.json 2>/dev/null || echo '{"results":[]}' > bandit-report.json
                '''
            }
        }

        stage('SAST - Gosec (Go)') {
            steps {
                sh '''
                    docker run --rm -v $(pwd)/auth-service:/app -w /app golang:1.25-alpine sh -c "go install github.com/securego/gosec/v2/cmd/gosec@latest && gosec -fmt=json -out=/app/gosec-report.json ./..." || true
                    cp auth-service/gosec-report.json gosec-report.json 2>/dev/null || echo '{}' > gosec-report.json
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                sh "mkdir -p ${env.WORKSPACE}/.trivy-cache"
                sh """
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  -v ${env.WORKSPACE}:/workspace \
                  -v ${env.WORKSPACE}/.trivy-cache:/root/.cache/trivy \
                  aquasec/trivy image \
                  --severity HIGH,CRITICAL \
                  --format json \
                  -o /workspace/trivy-auth.json \
                  --timeout 15m \
                  bookslib-auth-service
                """
            }
        }

        stage('Security Gate & Issue Creation') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                    script {
                        // Hitung kerentanan
                        def trivyCount = sh(script: """
                            python3 -c "import json; d = json.load(open('trivy-auth.json')); \
                            print(sum(len(r.get('Vulnerabilities', [])) for r in d.get('Results', [])))"
                        """, returnStdout: true).trim().toInteger()
                        
                        echo "Trivy findings: ${trivyCount}"

                        if (trivyCount > 0) {
                            echo "🚨 Ditemukan ${trivyCount} kerentanan! Melaporkan ke GitHub..."
                            
                            // Autentikasi GitHub CLI
                            sh 'echo "$GH_TOKEN" | gh auth login --with-token'
                            
                            // Buat Issue
                            sh "gh issue create --repo ${env.GITHUB_REPO} --title 'Security Alert: ${trivyCount} vulnerabilities found' --body 'Trivy scan mendeteksi ${trivyCount} HIGH/CRITICAL vulnerabilities di auth-service. Pipeline dihentikan!'"
                            
                            // BLOCK PIPELINE (Tidak lanjut ke Deploy)
                            error("Pipeline GAGAL: Aplikasi tidak aman untuk di-deploy!")
                        } else {
                            echo "✅ Tidak ada kerentanan ditemukan. Melanjutkan ke Deploy."
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: 'bookslib-env', variable: 'ENV_FILE')]) {
                    sh '''
                        cp $ENV_FILE .env
                        docker compose down || true
                        docker compose up -d
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed! Silakan periksa GitHub Issues untuk detail kerentanan.'
        }
    }
}

