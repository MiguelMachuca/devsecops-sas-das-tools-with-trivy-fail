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

    stage('DAST - OWASP ZAP Scan con Reporte') {
        agent {
            docker {
                image 'zaproxy/zap-stable:latest'
                args '-v $WORKSPACE:/zap/wrk:rw --network=host'  
            }
        }
        steps {
            script {
                // Cambiar al directorio montado que ZAP requiere
                sh '''
                    cd /zap/wrk
                    pwd
                    ls -la
                    zap-baseline.py -t ${STAGING_URL} -J zap-report.json -r zap-report.html -I
                    ls -la
                '''
                
                // Verificar que los reportes se crearon
                sh 'ls -la /zap/wrk/zap-report.* || echo "No se encontraron reportes en /zap/wrk"'
            }
        }
        post {
            always {
                // Archivar los reportes - ahora estarán en el workspace gracias al volumen
                archiveArtifacts artifacts: 'zap-report.*', allowEmptyArchive: true
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