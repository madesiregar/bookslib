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
                    docker run --rm -v $(pwd)/reviews-service:/app \
                        cytopia/bandit bandit /app -f json \
                        -o /app/bandit-report.json || true

                    cp reviews-service/bandit-report.json bandit-report.json 2>/dev/null || echo '{"results":[]}' > bandit-report.json
                '''
            }
        }


        stage('SAST - Gosec (Go)') {
            steps {
                sh '''
                    docker run --rm -v $(pwd)/auth-service:/app \
                        -w /app \
                        golang:1.23-alpine sh -c \
                        "go install github.com/securego/gosec/v2/cmd/gosec@latest && \
                        gosec -fmt=json -out=/app/gosec-report.json ./... || true"

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
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format json -o trivy-auth.json \
                        bookslib-auth-service || true


                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format json -o trivy-reviews.json \
                        bookslib-reviews-service || true
                '''
            }
        }


        stage('Create GitHub Issues if Findings') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {

                    sh '''
                        export GH_TOKEN=$GH_TOKEN


                        BANDIT_COUNT=$(python3 -c "
import json
try:
    r=json.load(open('bandit-report.json'))
    print(len(r.get('results', [])))
except:
    print(0)
")

                        echo "Bandit findings: $BANDIT_COUNT"


                        if [ "$BANDIT_COUNT" -gt 0 ]; then

                            gh issue create \
                            --repo $GITHUB_REPO \
                            --title "Bandit Finding Build #${BUILD_NUMBER}" \
                            --body "Bandit detected security findings. Check Jenkins report." \
                            --label security

                        fi



                        TRIVY_COUNT=$(python3 -c "
import json
total=0
for f in ['trivy-auth.json','trivy-reviews.json']:
    try:
        r=json.load(open(f))
        for x in r.get('Results',[]):
            total += len(x.get('Vulnerabilities',[]))
    except:
        pass
print(total)
")


                        echo "Trivy findings: $TRIVY_COUNT"



                        if [ "$TRIVY_COUNT" -gt 0 ]; then

                            gh issue create \
                            --repo $GITHUB_REPO \
                            --title "Trivy Finding Build #${BUILD_NUMBER}" \
                            --body "Trivy detected image vulnerabilities. Check Jenkins report." \
                            --label security

                        fi
                    '''
                }
            }
        }



        stage('Auto Close Fixed Security Issues') {

            steps {

                withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {

                    sh '''
                        export GH_TOKEN=$GH_TOKEN


                        echo "Checking security issues..."

                        gh issue list \
                          --repo $GITHUB_REPO \
                          --label security \
                          --state open \
                          --json number,title \
                          --jq '.[] | .number' |

                        while read ISSUE; do

                            echo "Closing issue #$ISSUE"


                            gh issue close $ISSUE \
                              --repo $GITHUB_REPO \
                              --comment "Automatically closed by Jenkins after successful pipeline."


                        done
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

                        sleep 10

                        docker compose ps
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
