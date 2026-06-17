pipeline {
    agent any

    environment {
        COMPOSE_FILE = 'docker-compose.yaml'
        GITHUB_REPO = 'madesiregar/bookslib' // Pastikan ini benar
        // Pastikan 'github-token' sudah diset di Manage Jenkins > Credentials
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SAST - Bandit (Python)') {
            steps {
                sh 'docker run --rm -v $(pwd)/reviews-service:/app cytopia/bandit -r /app -f json -o /app/bandit-report.json || true'
            }
        }

        stage('SAST - Gosec (Go)') {
            steps {
                sh 'docker run --rm -v $(pwd)/auth-service:/app -w /app golang:1.25-alpine sh -c "go install github.com/securego/gosec/v2/cmd/gosec@latest && gosec -fmt=json -out=/app/gosec-report.json ./..." || true'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                sh """
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  -v ${env.WORKSPACE}:/workspace \
                  aquasec/trivy image \
                  --severity HIGH,CRITICAL \
                  --format json \
                  -o /workspace/trivy-auth.json \
                  bookslib-auth-service || true
                """
            }
        }

        stage('Security Gate & Issue Creation') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                    script {
                        def trivyCount = sh(script: """
                            python3 -c "import json; d = json.load(open('trivy-auth.json')); \
                            print(sum(len(r.get('Vulnerabilities', [])) for r in d.get('Results', [])))"
                        """, returnStdout: true).trim().toInteger()

                        if (trivyCount > 0) {
                            echo "🚨 ${trivyCount} kerentanan ditemukan! Melapor ke GitHub via API..."
                            
                            // Menggunakan cURL untuk bypass 'gh not found'
                            sh """
                                curl -L -X POST \
                                -H "Accept: application/vnd.github+json" \
                                -H "Authorization: Bearer ${GH_TOKEN}" \
                                -H "X-GitHub-Api-Version: 2022-11-28" \
                                https://api.github.com/repos/${env.GITHUB_REPO}/issues \
                                -d '{"title":"Security Alert: ${trivyCount} Vulnerabilities Found", "body":"Pipeline dihentikan karena Trivy mendeteksi ${trivyCount} HIGH/CRITICAL vulnerabilities di auth-service."}'
                            """
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
                        docker compose down
                        docker compose up -d
                    '''
                }
            }
        }
    }
}
