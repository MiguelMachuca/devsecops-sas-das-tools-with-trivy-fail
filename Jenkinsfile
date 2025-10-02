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
                sh '''
                    cd /zap/wrk
                    # Generar reportes en JSON, HTML y XML
                    zap-baseline.py -t ${STAGING_URL} -J zap-report.json -r zap-report.html -x zap-report.xml -I
                    # Copiar los reportes al workspace principal
                    cp zap-report.* $WORKSPACE/ || true
                '''
            }
        }
        post {
            always {
                // Archivar todos los reportes (json, html, xml)
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