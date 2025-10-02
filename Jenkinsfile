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
                args '--network=host'  // ELIMINADO el volumen
            }
        }
        steps {
            script {
                // Ejecutar ZAP y manejar el código de salida
                // ZAP retorna 2 cuando encuentra warnings, pero queremos continuar
                def zapExitCode = sh(
                    script: "zap-baseline.py -t ${STAGING_URL} -J zap-report.json -r zap-report.html || true",
                    returnStatus: true
                )
                
                echo "ZAP ejecutado con código de salida: ${zapExitCode}"
                echo "Los reportes deberían estar en:"
                sh 'pwd && ls -la zap-report.* || echo "No se encontraron reportes"'
            }
        }
        post {
            always {
                // Archivar los reportes independientemente del resultado de ZAP
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
