pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME = 'koya-clinic'
        APP_PORT = '8010'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Docker') {
            steps {
                sh '''
                    docker version
                    docker compose version
                    docker compose config
                '''
            }
        }

        
stage('Backend Tests') {
    steps {
        sh '''
            set -e

            TEST_CONTAINER="koya-clinic-test-${BUILD_NUMBER}"

            cleanup() {
                docker rm -f "$TEST_CONTAINER" >/dev/null 2>&1 || true
            }

            trap cleanup EXIT

            docker run -d \
              --name "$TEST_CONTAINER" \
              -e API_KEY=test-secret \
              -e DATABASE_URL=sqlite:///./test.db \
              python:3.12-slim sleep infinity

            docker cp "$WORKSPACE/." "$TEST_CONTAINER:/app"

            docker exec -w /app "$TEST_CONTAINER" \
              sh -c "pip install -r requirements.txt pytest==8.3.3 httpx==0.27.2 && pytest -v"
        '''
    }
}


        stage('Build') {
            steps {
                withCredentials([
                    string(credentialsId: 'koya-api-key', variable: 'API_KEY')
                ]) {
                    sh 'docker compose build'
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'koya-api-key', variable: 'API_KEY')
                ]) {
                    sh 'docker compose up -d'
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                        if curl -fsS http://localhost:8010/docs >/dev/null; then
                            echo "Koya Clinic deployment successful"
                            exit 0
                        fi
                        sleep 5
                    done
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'Koya Clinic CI/CD completed successfully'
        }
        failure {
            echo 'Koya Clinic CI/CD failed'
        }
    }
}
