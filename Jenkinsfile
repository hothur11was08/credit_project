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

        stage('Deploy') {
            steps {
                sh 'docker-compose up -d'
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
                sh 'curl -s ${API_URL}/schema'
                sh 'curl -s -X POST ${API_URL}/performance/compute -H "Content-Type: application/json" -d \'{"window_hours":24}\''
                sh 'curl -s -X POST ${API_URL}/drift/compute -H "Content-Type: application/json" -d \'{"window_hours":24}\''
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
            sh 'docker-compose ps'
        }
    }
}
