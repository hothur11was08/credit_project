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

        stage('Deploy & Health Check') {
            steps {
                // Start containers in detached mode
                sh 'docker-compose up -d'
                // Wait until API is ready
                sh '''
                  for i in {1..20}; do
                    curl -s ${API_URL}/schema && exit 0
                    sleep 2
                  done
                  echo "API did not start in time" && exit 1
                '''
            }
        }

        stage('Sanity Checks') {
            steps {
                // Run sanity checks but don’t fail pipeline if endpoints error
                sh 'curl -s ${API_URL}/schema || true'
                sh 'curl -s -X POST ${API_URL}/performance/compute -H "Content-Type: application/json" -d \'{"window_hours":24}\' || true'
                sh 'curl -s -X POST ${API_URL}/drift/compute -H "Content-Type: application/json" -d \'{"window_hours":24}\' || true'
            }
        }

        stage('Train Model') {
            steps {
                // Run training script if present
                sh 'docker-compose run api python scripts/train.py || true'
                archiveArtifacts artifacts: 'models/*.h5', fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Drift Detection') {
            steps {
                // Run drift detection script if present
                sh 'docker-compose run api python scripts/drift_detection.py || true'
                archiveArtifacts artifacts: 'drift_reports/*.png', fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Archive Metrics') {
            steps {
                sh 'curl -s ${API_URL}/performance/latest > performance_latest.json || true'
                sh 'curl -s ${API_URL}/drift/latest > drift_latest.json || true'
                archiveArtifacts artifacts: 'performance_latest.json, drift_latest.json', fingerprint: true, allowEmptyArchive: true
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
