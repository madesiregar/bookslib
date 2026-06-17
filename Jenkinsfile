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
                    rm -f bandit-report.json

                    docker run --rm \
                        -v $(pwd)/reviews-service:/app \
                        cytopia/bandit \
                        -r /app \
                        -f json \
                        -o /app/bandit-report.json || true

                    cp reviews-service/bandit-report.json bandit-report.json 2>/dev/null || echo '{"results":[]}' > bandit-report.json

                    echo "===== BANDIT RESULT ====="
                    cat bandit-report.json
                '''
            }
        }

        stage('SAST - Gosec (Go)') {
            steps {
                sh '''
                    rm -f gosec-report.json

                    docker run --rm \
                        -v $(pwd)/auth-service:/app \
                        -w /app \
                        securego/gosec \
                        -fmt=json \
                        -out=gosec-report.json \
                        ./... || true

                    cp auth-service/gosec-report.json gosec-report.json 2>/dev/null || echo '{}' > gosec-report.json
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker compose build
                '''
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v $(pwd):/workspace \
                        aquasec/trivy image \
                        --severity HIGH,CRITICAL \
                        --format json \
                        -o /workspace/trivy-auth.json \
                        bookslib-auth-service || true

                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v $(pwd):/workspace \
                        aquasec/trivy image \
                        --severity HIGH,CRITICAL \
                        --format json \
                        -o /workspace/trivy-reviews.json \
                        bookslib-reviews-service || true
                '''
            }
        }

        stage('Create GitHub Security Issues') {
            steps {
                withCredentials([
                    string(credentialsId: 'github-token', variable: 'GH_TOKEN')
                ]) {
                    sh '''
                        export GH_TOKEN=$GH_TOKEN

                        echo "$GH_TOKEN" | gh auth login --with-token || true
                        gh auth status || true

                        echo "Calculating Bandit findings..."

                        BANDIT_COUNT=$(python3 - <<'EOF'
import json

try:
    with open("bandit-report.json") as f:
        data = json.load(f)
    print(len(data.get("results", [])))
except:
    print(0)
EOF
)

                        echo "Bandit findings: $BANDIT_COUNT"

                        if [ "$BANDIT_COUNT" -gt 0 ]; then
                            gh issue create \
                                --repo "$GITHUB_REPO" \
                                --title "Security Finding - Bandit Build #${BUILD_NUMBER}" \
                                --body "Bandit detected $BANDIT_COUNT issue(s)." \
                                --label security || true
                        fi

                        echo "Calculating Trivy findings..."

                        TRIVY_COUNT=$(python3 - <<'EOF'
import json

total = 0

for file in ["trivy-auth.json", "trivy-reviews.json"]:
    try:
        with open(file) as f:
            data = json.load(f)

        for r in data.get("Results", []):
            total += len(r.get("Vulnerabilities", []) or [])
    except:
        pass

print(total)
EOF
)

                        echo "Trivy findings: $TRIVY_COUNT"

                        if [ "$TRIVY_COUNT" -gt 0 ]; then
                            gh issue create \
                                --repo "$GITHUB_REPO" \
                                --title "Security Finding - Trivy Build #${BUILD_NUMBER}" \
                                --body "Trivy detected $TRIVY_COUNT vulnerability(ies)." \
                                --label security || true
                        fi
                    '''
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

                        docker compose down || true
                        docker compose up -d

                        sleep 15
                        docker compose ps
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.json', allowEmptyArchive: true
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
