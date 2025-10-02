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
                args '-v $WORKSPACE:/zap/wrk:rw --network=host'  
            }
        }
        steps {
            // Crear el directorio para los reportes primero
            sh 'mkdir -p /zap/wrk/zap-reports'
            
            // Ejecutar ZAP y generar reportes
            sh 'zap-baseline.py -t ${STAGING_URL} -J /zap/wrk/zap-reports/zap-report.json -r /zap/wrk/zap-reports/zap-report.html'
            
            // Verificar que los archivos se crearon
            sh 'ls -la /zap/wrk/zap-reports/'
        }
        post {
            always {
                // Archivar los reportes desde el workspace (no desde el volumen del contenedor)
                archiveArtifacts artifacts: 'zap-reports/*', allowEmptyArchive: true
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
