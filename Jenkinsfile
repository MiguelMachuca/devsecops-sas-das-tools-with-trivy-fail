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
                def jenkinsWorkspace = pwd()
                
                docker.image('bridgecrew/checkov:latest').inside("--entrypoint=''") {
                    
                    sh '''
                      checkov -f docker-compose.yml -f Dockerfile \
                        --soft-fail \
                        --output json --output-file-path results-checkov.json \
                        --output junitxml --output-file-path results-checkov
                    '''

                    sh 'ls -la'
                    
   
                    sh """
                        if [ -f "results-checkov.json" ]; then
                            cp results-checkov.json ${jenkinsWorkspace}/checkov-results.json
                        elif [ -d "results-checkov" ]; then
                            cp -r results-checkov/* ${jenkinsWorkspace}/
                        fi
                    """
                }
                
                // Publicar resultados JUnit si existen
                script {
                    if (fileExists('results-checkov/results_junitxml.xml')) {
                        junit skipPublishingChecks: true, testResults: 'results-checkov/results_junitxml.xml'
                    } else if (fileExists('results_junitxml.xml')) {
                        junit skipPublishingChecks: true, testResults: 'results_junitxml.xml'
                    } else {
                        echo 'WARNING: No se encontraron archivos de resultados JUnit para publicar'
                    }
                }
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