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
                        checkov -f docker-compose.yml -f Dockerfile \
                          --soft-fail \
                          --output checkov --output-file-path checkov-report.json \
                          --output checkov --output-file-path checkov-report.xml
                    '''
                    sh 'ls -la checkov-report.*'
                }
            }
        }
        post {
            always {                
                archiveArtifacts artifacts: 'results_checkov.*', allowEmptyArchive: true
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