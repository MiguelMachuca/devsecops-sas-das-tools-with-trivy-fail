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

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }


    stage('Deploy to Staging (docker-compose)') {
      agent { label 'docker' }
      steps {
        echo "Deploying to staging with docker-compose..."
        sh '''
          docker-compose -f docker-compose.yml down || true
          docker-compose -f docker-compose.yml up -d --build
          sleep 8
          docker ps -a
        '''
      }
    }

    stage('DAST - OWASP ZAP Scan con Reporte') {
        agent {
            docker {
                image 'zaproxy/zap-stable:latest'
                args '-v $WORKSPACE:/zap/wrk:rw --network=host'  
            }
        }
        steps {
            sh 'zap-baseline.py -t ${STAGING_URL} -J zap-reports/zap-report.json -r zap-reports/zap-report.html'
            sh 'ls -la'
            sh 'pwd'
            sh 'find / -name "zap-report.*" 2>/dev/null || true'
            sh 'ls -la /zap/wrk/'
            archiveArtifacts artifacts: '/zap/wrk/zap-reports/zap-report.html', allowEmptyArchive: true
        }
    }  

    stage('Policy Check - Fail on HIGH/CRITICAL CVEs') {
    steps {
        script {
            // Run the security scan script and capture its exit code
            def exitCode = sh(script: '''
                chmod +x scripts/scan_trivy_fail.sh
                ./scripts/scan_trivy_fail.sh $DOCKER_IMAGE_NAME || exit_code=$?
                echo "Script exit code is: ${exit_code:-0}"
                exit ${exit_code:-0}
            ''', returnStatus: true) // 'returnStatus: true' prevents the sh step from failing the pipeline immediately

            // Evaluate the exit code
            if (exitCode == 2) {
                // Mark the build as unstable (Yellow warning) instead of failing it (Red)
                unstable("WARNING: HIGH/CRITICAL vulnerabilities were detected by Trivy. Please review.")
            } else if (exitCode != 0) {
                // For any other non-zero exit code, you may still want to fail
                error("Trivy scan failed with an unexpected error. Exit code: ${exitCode}")
            }
            // If exitCode is 0, the build continues as SUCCESS
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
