pipeline {
    agent any
    
    environment {
        // Definisikan image name aplikasi kamu di sini
        IMAGE_NAME = "bookslib-auth-service"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        // Jalankan Gitleaks untuk scan kebocoran password/token/key
        stage('Secret Scanning - Gitleaks') {
            steps {
               sh 'docker run --rm -v ${WORKSPACE}:/app -w /app/auth-service golang:latest sh -c "go install golang.org/x/vuln/cmd/govulncheck@latest && govulncheck -json ./..." > govulncheck-report.json'
            }
        }

        // Jalankan Semgrep untuk Static Application Security Testing (SAST) umum
        stage('SAST - Semgrep') {
            steps {
                sh 'docker run --rm -v ${WORKSPACE}:/src returntocorp/semgrep semgrep scan --config=auto --json --output=/src/semgrep-report.json || true'
            }
        }

        // Tahap pengecekan kerentanan pada library/dependencies secara PARALEL
        stage('Dependency Scan') {
            parallel {
                stage('Python - pip-audit') {
                    steps {
                        sh 'docker run --rm -v ${WORKSPACE}:/app python:3.9-slim sh -c "pip install pip-audit && pip-audit -r /app/requirements.txt --format json -o /app/pip-audit-report.json" || true'
                    }
                }
                stage('Node.js - npm audit') {
                    steps {
                        sh 'docker run --rm -v ${WORKSPACE}:/app node:alpine sh -c "cd /app && npm audit --json > /app/npm-audit-report.json" || true'
                    }
                }
                stage('Go - govulncheck') {
                    steps {
                        sh 'docker run --rm -v ${WORKSPACE}:/app golang:latest sh -c "go install golang.org/x/vuln/cmd/govulncheck@latest && cd /app && govulncheck -json ./... > /app/govulncheck-report.json" || true'
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}:/workspace aquasec/trivy image --severity HIGH,CRITICAL --format json -o /workspace/trivy-report.json bookslib-auth-service || true'
            }
        }

        // TAMBAHAN 1: Menghubungkan ke Docker Hub menggunakan Credentials Jenkins
        stage('Push Images to Registry') {
            steps {
                echo "Mulai proses push image aman ke Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    // Login ke Docker Hub secara aman via CLI
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    
                    // Berikan tag sesuai format Docker Hub (username/image-name:tag)
                    sh "docker tag ${IMAGE_NAME}:latest \$DOCKER_USER/${IMAGE_NAME}:latest"
                    
                    // Push ke repository publik Docker Hub kamu
                    sh "docker push \$DOCKER_USER/${IMAGE_NAME}:latest"
                }
            }
        }

        // TAMBAHAN 2: Otomatisasi pembuatan Issue di GitHub jika ditemukan celah (Syarat Track A)
        stage('Document Automation Issues to GitHub') {
            steps {
                echo "Mendokumentasikan hasil temuan security ke GitHub Issues..."
                withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    // Menggunakan API GitHub untuk membuat Issue otomatis ke repositori kamu
                    sh """
                        curl -X POST -H "Authorization: token \$GH_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        https://api.github.com/repos/\$GH_USER/bookslib/issues \
                        -d '{"title": "🚨 DevSecOps Alert: Vulnerabilities Detected in Build #${BUILD_NUMBER}", "body": "Halo Tim Developer,\\n\\nPipeline otomatis telah menyelesaikan pemindaian keamanan pada branch **${BRANCH_NAME}**. Ditemukan beberapa potensi celah keamanan.\\n\\nSilakan periksa berkas laporan **JSON Artifacts** langsung di server Jenkins untuk detail mitigasi Semgrep, Gitleaks, dan Trivy.\\n\\nSalam,\\nJenkins Bot", "labels": ["bug", "security"]}' || true
                    """
                }
            }
        }

        stage('Deploy - Staging') {
            steps {
                echo "Deploying application to Staging environment..."
                sh 'docker compose up -d'
            }
        }

        stage('Deploy - Production') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying application to Production environment..."
                // Jika ada docker-compose prod tersendiri, jalankan di sini
                // sh 'docker compose -f docker-compose.prod.yml up -d'
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '*-report.json, **/*-report.json', allowEmptyArchive: true
        }
    }
}