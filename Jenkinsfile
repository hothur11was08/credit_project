pipeline {
    agent any

    environment {
        API_URL = "http://localhost:8000"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'docker-compose build'
            }
        }

        stage('Deploy & Test') {
            steps {
                // Run containers in foreground, exit when API container finishes
                sh 'docker-compose up --abort-on-container-exit --exit-code-from api'
            }
        }

        stage('Train Model') {
            steps {
                // Run training script inside API container
                sh 'docker-compose run api python scripts/train.py'
                archiveArtifacts artifacts: 'models/*.h5', fingerprint: true
            }
        }

        stage('Drift Detection') {
            steps {
                // Run drift detection script
                sh 'docker-compose run api python scripts/drift_detection.py'
                archiveArtifacts artifacts: 'drift_reports/*.png', fingerprint: true
            }
        }

        stage('Archive Metrics') {
            steps {
                sh 'curl -s ${API_URL}/performance/latest > performance_latest.json'
                sh 'curl -s ${API_URL}/drift/latest > drift_latest.json'
                archiveArtifacts artifacts: 'performance_latest.json, drift_latest.json', fingerprint: true
            }
        }
    }

    post {
        always {
            // Clean up containers to avoid conflicts next run
            sh 'docker-compose down'
        }
    }
}
