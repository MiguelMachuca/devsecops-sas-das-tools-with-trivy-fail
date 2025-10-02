pipeline {
  agent any

  environment {
    DOCKER_REGISTRY = "docker.io"
    DOCKER_CREDENTIALS = "docker-registry-credentials"
    GIT_CREDENTIALS = "git-credentials"
    DOCKER_IMAGE_NAME = "mangelmy/devsecops-app:latest"
    SSH_CREDENTIALS = "ssh-deploy-key"
    STAGING_URL = "http://localhost:3000"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    ansiColor('xterm')
  }

  stages {

    stage('IaC Scan - Checkov') {
        agent any
        steps {
            script {
                docker.image('bridgecrew/checkov:latest').inside("--entrypoint=''") {
                  sh '''
                      # Limpiar archivos previos
                      rm -f checkov-report.* checkov-scan-results.*
                      
                      # Ejecutar Checkov - This works and generates files
                      checkov -f docker-compose.yml -f Dockerfile \
                        --soft-fail \
                        --output json --output-file-path checkov-results \
                        --output junitxml --output-file-path checkov-results

                      # Copy from the correct location: current directory
                      cp checkov-results_json.json checkov-scan-results.json
                      cp checkov-results_junitxml.xml checkov-scan-results.xml
                  '''
                }
            }
        }
        post {
            always {
                junit testResults: 'checkov-scan-results.xml', allowEmptyResults: true
                
                archiveArtifacts artifacts: 'checkov-scan-results.json, checkov-scan-results.xml', allowEmptyArchive: true
            }
        }
    }
  }

  post {
    always {
      echo "Pipeline finished. Collecting artifacts..."
    }
    failure {
      echo "Pipeline failed!"
    }
  }
}