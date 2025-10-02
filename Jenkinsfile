pipeline {
  agent any

  environment {
    DOCKER_REGISTRY = "docker.io"
    DOCKER_CREDENTIALS = "docker-registry-credentials"
    GIT_CREDENTIALS = "git-credentials"
    DOCKER_IMAGE_NAME = "mangelmy/devsecops-app:latest"
    //DOCKER_IMAGE_NAME = "${env.DOCKER_REGISTRY}/devsecops-labs/app:latest"
    SSH_CREDENTIALS = "ssh-deploy-key"
    STAGING_URL = "http://localhost:3000"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    ansiColor('xterm')
  }

  stages {

    stage('DAST - OWASP ZAP Scan con Reporte') {
        agent {
            docker {
                image 'zaproxy/zap-stable:latest'
                args '--network=host'  
            }
        }
        steps {
            script {
                // Cambiar al directorio que ZAP espera o usar rutas relativas
                sh '''
                    pwd
                    ls -la
                    zap-baseline.py -t ${STAGING_URL} -J ./zap-report.json -r ./zap-report.html -I
                    ls -la
                '''
                
                // Buscar reportes en todo el workspace
                sh 'find . -name "*zap-report*" -type f 2>/dev/null || echo "No se encontraron reportes"'

                sh '''
                    echo "=== Diagnóstico del sistema ==="
                    pwd
                    echo "=== Contenido del directorio actual ==="
                    ls -la
                    echo "=== Buscando reportes ZAP ==="
                    find . -name "*.json" -o -name "*.html" 2>/dev/null | grep -v node_modules || echo "No se encontraron archivos de reporte"
                    echo "=== Espacio en disco ==="
                    df -h
                '''

                sh 'whoami && id'                
            }
        }
        post {
            always {
                // Archivar con patrón más amplio
                archiveArtifacts artifacts: '**/zap-report.*', allowEmptyArchive: true
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
