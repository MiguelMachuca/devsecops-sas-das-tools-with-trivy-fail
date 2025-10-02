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
          script {
              // Crear directorio en la ubicación correcta
              sh 'mkdir -p /zap/wrk/zap-reports'
              
              // Cambiar al directorio de trabajo correcto antes de ejecutar ZAP
              sh 'cd /zap/wrk'
              
              // Ejecutar ZAP con rutas relativas al directorio de trabajo
              sh 'zap-baseline.py -t ${STAGING_URL} -J zap-reports/zap-report.json -r zap-reports/zap-report.html'
              
              // Verificar que los archivos se crearon correctamente
              sh 'ls -la /zap/wrk/zap-reports/'
              
              // También verificar desde la perspectiva del workspace
              sh 'ls -la zap-reports/ || echo "No existe en workspace relativo"'
          }
      }
      post {
          always {
              // Archivar usando patrón que busca en todo el workspace :cite[1]:cite[6]
              archiveArtifacts artifacts: '**/zap-report.*', allowEmptyArchive: true
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
