pipeline {
agent any

```
environment {
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
            docker run --rm \
              -v $(pwd)/reviews-service:/app \
              cytopia/bandit \
              -r /app \
              -f json \
              -o /app/bandit-report.json || true
            '''
        }
    }

    stage('SAST - Gosec (Go)') {
        steps {
            sh '''
            docker run --rm \
              -v $(pwd)/auth-service:/app \
              -w /app \
              golang:1.25-alpine \
              sh -c "
              go install github.com/securego/gosec/v2/cmd/gosec@latest &&
              /root/go/bin/gosec -fmt=json -out=/app/gosec-report.json ./...
              " || true
            '''
        }
    }

    stage('Build Docker Images') {
        steps {
            sh '''
            docker compose build
            docker images
            '''
        }
    }

    stage('Image Scan - Trivy') {
        steps {
            sh '''
            rm -f trivy-auth.json

            docker run --rm \
              -v /var/run/docker.sock:/var/run/docker.sock \
              -v $(pwd):/workspace \
              aquasec/trivy image \
              --skip-version-check \
              --severity HIGH,CRITICAL \
              --format json \
              -o /workspace/trivy-auth.json \
              bookslib-auth-service

            echo "=== VERIFY TRIVY REPORT ==="
            ls -lah trivy-auth.json
            '''
        }
    }

    stage('Security Gate & Issue Creation') {
        steps {
            withCredentials([
                string(credentialsId: 'github-token', variable: 'GH_TOKEN')
            ]) {
                script {

                    sh 'test -f trivy-auth.json'

                    def trivyCount = sh(
                        script: '''
                        python3 - <<EOF
```

import json

with open("trivy-auth.json") as f:
data = json.load(f)

count = 0
for r in data.get("Results", []):
count += len(r.get("Vulnerabilities", []))

print(count)
EOF
''',
returnStdout: true
).trim().toInteger()

```
                    echo "Total Vulnerabilities: ${trivyCount}"

                    if (trivyCount > 0) {

                        sh """
                        curl -L -X POST \
                          -H "Accept: application/vnd.github+json" \
                          -H "Authorization: Bearer ${GH_TOKEN}" \
                          -H "X-GitHub-Api-Version: 2022-11-28" \
                          https://api.github.com/repos/${env.GITHUB_REPO}/issues \
                          -d '{
                            "title":"Security Alert: ${trivyCount} Vulnerabilities Found",
                            "body":"Pipeline dihentikan kerana Trivy mendeteksi ${trivyCount} HIGH/CRITICAL vulnerabilities pada auth-service."
                          }'
                        """

                        error("Pipeline GAGAL: ditemukan ${trivyCount} vulnerability")
                    }

                    echo "Tidak ada vulnerability ditemukan"
                }
            }
        }
    }

    stage('Deploy') {
        steps {
            withCredentials([
                file(credentialsId: 'bookslib-env', variable: 'ENV_FILE')
            ]) {
                sh '''
                cp $ENV_FILE .env
                docker compose down
                docker compose up -d
                '''
            }
        }
    }
}

post {
    always {
        archiveArtifacts artifacts: '*.json', allowEmptyArchive: true
    }
}
```

}
