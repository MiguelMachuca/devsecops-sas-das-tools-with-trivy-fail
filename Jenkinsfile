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
                        # Ejecutar Checkov - generará archivos en directorio results-checkov/
                        checkov -f docker-compose.yml -f Dockerfile \
                          --soft-fail \
                          --output json --output-file-path checkov-results \
                          --output junitxml --output-file-path checkov-results
                        
                        # Verificar qué archivos se crearon
                        echo "=== Contenido del directorio results-checkov/ ==="
                        ls -la results-checkov/
                        
                        # Copiar y renombrar los archivos con nombres más descriptivos
                        cp results-checkov/results_json.json checkov-scan-results.json
                        cp results-checkov/results_junitxml.xml checkov-scan-results.xml
                        
                        # Verificar los archivos renombrados
                        echo "=== Archivos finales ==="
                        ls -la checkov-*.json checkov-*.xml
                    '''
                }
            }
        }
        post {
            always {
                // Publicar resultados de pruebas JUnit
                junit testResults: 'checkov-scan-results.xml', allowEmptyResults: true
                
                // Archivar todos los reportes de Checkov
                archiveArtifacts artifacts: 'checkov-*.json, checkov-*.xml, results-checkov/**', allowEmptyArchive: true
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