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
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/workspace aquasec/trivy image --severity HIGH,CRITICAL --format json -o /workspace/trivy-auth.json bookslib-auth-service || true
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/workspace aquasec/trivy image --severity HIGH,CRITICAL --format json -o /workspace/trivy-reviews.json bookslib-reviews-service || true
                '''
            }
        }

stage('Create GitHub Security Issues') {
    steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
            sh '''
                export GH_TOKEN=$GH_TOKEN
                echo "$GH_TOKEN" | gh auth login --with-token || true

                TRIVY_COUNT=$(python3 -c "
import json
total = 0
for f in ['trivy-auth.json', 'trivy-reviews.json']:
    try:
        d = json.load(open(f))
        for r in d.get('Results', []):
            total += len(r.get('Vulnerabilities', []) or [])
    except: pass
print(total)
")
                echo "Trivy findings: $TRIVY_COUNT"
                if [ "$TRIVY_COUNT" -gt 0 ]; then
                    gh issue create \
			--repo madesiregar/bookslib
                        --title "Security: $TRIVY_COUNT vulnerabilities found" \
                        --body "Trivy scan found $TRIVY_COUNT HIGH/CRITICAL vulnerabilities. Check trivy-auth.json and trivy-reviews.json for details."
                fi
            '''
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
            echo 'Pipeline failed!'
        }
    }
}
