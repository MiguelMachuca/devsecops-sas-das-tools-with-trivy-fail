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
                    zap-baseline.py -t ${STAGING_URL} -J zap-report.json -r zap-report.html -I
                '''
            }
        }
        post {
            always {
                // Busca en todo el workspace recursivamente
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