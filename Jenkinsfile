pipeline {
    agent any
    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'amanjolas'
        SONAR_HOST      = 'http://host.docker.internal:9000'
        SONAR_TOKEN     = credentials('sonar-token')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Kod alindi: ${env.GIT_COMMIT?.take(7)}"
            }
        }
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv || python -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }
        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    mkdir -p test-results
                    python -m pytest tests/test_app.py \
                        -v \
                        --tb=short \
                        --junit-xml=test-results/unit-tests.xml \
                        --cov=app \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing
                '''
            }
            post {
                always {
                    junit 'test-results/unit-tests.xml'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                sh """
                    docker run --rm \
                        -e SONAR_HOST_URL=${SONAR_HOST} \
                        -e SONAR_TOKEN=${SONAR_TOKEN} \
                        -v \$(pwd):/usr/src \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=techstore \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/**
                """
            }
        }
        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} -t ${DOCKER_IMAGE}:latest .
                """
                echo "Docker imaji olusturuldu"
            }
        }
        stage('Smoke Test') {
            steps {
                sh '''
                    docker stop techstore-app 2>/dev/null || true
                    docker rm techstore-app 2>/dev/null || true
                    docker run -d --name techstore-app -p 5000:5000 techstore-app:latest
                    sleep 10
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://host.docker.internal:5000/health)
                    if [ "$STATUS" != "200" ]; then
                        echo "Smoke test basarisiz! HTTP: $STATUS"
                        exit 1
                    fi
                    echo "Smoke test gecildi"
                '''
            }
        }
    }
    post {
        always {
            sh "docker image prune -f || true"
            cleanWs()
        }
        success {
            echo "Pipeline basariyla tamamlandi!"
        }
        failure {
            echo "Pipeline basarisiz!"
        }
    }
}