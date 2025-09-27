pipeline {
  agent any

  environment {
    DOCKER_REGISTRY = "docker.io"
    DOCKER_CREDENTIALS = "docker-registry-credentials"
    GIT_CREDENTIALS = "git-credentials"
    DOCKER_IMAGE_NAME = "mangelmy/devsecops-app:latest"
    SSH_CREDENTIALS = "ssh-deploy-key"
    STAGING_URL = "http://localhost:3000"
    REPORTS_DIR = "${WORKSPACE}/security-reports"
    BUILD_NUMBER = "${env.BUILD_NUMBER}"
    JOB_NAME = "${env.JOB_NAME}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    ansiColor('xterm')
  }

  stages {

    stage('Preparación Workspace') {
      steps {
        echo "Preparando directorio para reportes..."
        sh '''
          mkdir -p ${REPORTS_DIR}
          mkdir -p ${REPORTS_DIR}/sast
          mkdir -p ${REPORTS_DIR}/sca
          mkdir -p ${REPORTS_DIR}/container-scan
          mkdir -p ${REPORTS_DIR}/dast
          chmod -R 755 ${REPORTS_DIR}
          ls -la ${REPORTS_DIR}
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('SAST - Semgrep') {
      agent {
        docker { 
          image 'returntocorp/semgrep:latest'
          args '-v ${REPORTS_DIR}:/reports -u root'  
        }
      }
      steps {
        echo "Running Semgrep (SAST)..."
        sh '''
          semgrep --config=auto --json --output /reports/sast/semgrep-results.json . || true
          # Usar --sarif en lugar de --html (más compatible)
          semgrep --config=auto --sarif --output /reports/sast/semgrep-results.sarif . || true
          echo "SAST completado - Reportes guardados en ${REPORTS_DIR}/sast/"
        '''
      }
      post {
        always {
          script {
            echo "Archivando reportes SAST..."
            archiveArtifacts artifacts: 'security-reports/sast/*', allowEmptyArchive: true
          }
        }
      }
    }

    stage('SCA - Dependency Check') {
      agent {
        docker { 
          image 'owasp/dependency-check:latest'
          args '--network=host -v ${REPORTS_DIR}:/reports -u root'  
        }
      }
      steps {
        echo "Running SCA / Dependency-Check..."
        sh '''
          mkdir -p /reports/sca
          timeout 600 dependency-check --project "${JOB_NAME}-${BUILD_NUMBER}" --scan . --format JSON --out /reports/sca/dependency-check-report.json || true
          dependency-check --project "${JOB_NAME}-${BUILD_NUMBER}" --scan . --format HTML --out /reports/sca/dependency-check-report.html || true
          echo "SCA completado"
        '''
      }
      post {
        always {
          script {
            echo "Archivando reportes SCA..."
            archiveArtifacts artifacts: 'security-reports/sca/*', allowEmptyArchive: true
          }
        }
      }
    }

    stage('Build') {
      agent { 
        docker { 
          image 'node:18-alpine'
          args '-u root -w /workspace -v ${WORKSPACE}:/workspace'  
        }
      }
      steps {
        echo "Building app (npm install and tests)..."
        sh '''
          cd src
          npm install --no-audit --no-fund
          if [ -f package.json ]; then
            mkdir -p ${REPORTS_DIR}/tests
            npm test --silent -- --reporter=json --outputFile=${REPORTS_DIR}/tests/test-results.json || echo "Tests fallaron pero continuamos"
          fi
        '''
      }
    }

    stage('Docker Build & Trivy Scan') {
      agent { 
        docker { 
          image 'docker:latest' 
          args '-u root -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}:/workspace -w /workspace'
        }
      }
      steps {
        echo "Building Docker image..."
        sh '''
          docker build -t ${DOCKER_IMAGE_NAME} -f Dockerfile .
        '''
        echo "Scanning image with Trivy..."
        sh '''
          mkdir -p ${REPORTS_DIR}/container-scan
          # Instalar Trivy
          apk add --no-cache trivy || true
          # Scan completo con reportes múltiples
          trivy image --format json --output ${REPORTS_DIR}/container-scan/trivy-report.json ${DOCKER_IMAGE_NAME} || true
          trivy image --format table --output ${REPORTS_DIR}/container-scan/trivy-report.txt ${DOCKER_IMAGE_NAME} || true
          trivy image --severity HIGH,CRITICAL --exit-code 0 ${DOCKER_IMAGE_NAME} || true
          echo "Trivy scan completado"
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'security-reports/container-scan/*', allowEmptyArchive: true
        }
      }
    }

    stage('Push Image (optional)') {
      when {
        expression { return env.DOCKER_REGISTRY != null && env.DOCKER_REGISTRY != "" }
      }
      steps {
        echo "Pushing image to registry ${DOCKER_REGISTRY}..."
        withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_IMAGE_NAME}
            docker logout ${DOCKER_REGISTRY}
          '''
        }
      }
    }

    stage('Deploy to Staging') {
      agent { 
        docker {
          image 'docker/compose:latest'
          args '-u root -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}:/workspace -w /workspace'
        }
      }
      steps {
        echo "Deploying to staging with docker-compose..."
        sh '''
          docker-compose -f docker-compose.yml down || true
          docker-compose -f docker-compose.yml up -d --build
          sleep 15
          docker ps -a
          # Verificar que la aplicación esté corriendo
          curl -f ${STAGING_URL} || echo "La aplicación podría no estar lista aún"
        '''
      }
    }

    stage('DAST - OWASP ZAP scan') {
      agent { 
        docker { 
          image 'owasp/zap2docker-stable:latest'
          args '--network host -v ${REPORTS_DIR}:/zap/reports -u root'  
        }
      }
      steps {
        echo "Running DAST (OWASP ZAP) against ${STAGING_URL} ..."
        sh '''
          mkdir -p /zap/reports/dast
          zap-baseline.py -t ${STAGING_URL} -r /zap/reports/dast/zap-report.html || true
          # Generar reporte JSON también
          zap-baseline.py -t ${STAGING_URL} -J /zap/reports/dast/zap-report.json || true
          echo "DAST completado"
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'security-reports/dast/*', allowEmptyArchive: true
        }
      }
    }

    stage('Policy Check - Fail on HIGH/CRITICAL CVEs') {
      agent {
        docker { 
          image 'docker:latest'
          args '-u root -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}:/workspace -w /workspace'
        }
      }
      steps {
        sh '''
          chmod +x scripts/scan_trivy_fail.sh
          ./scripts/scan_trivy_fail.sh ${DOCKER_IMAGE_NAME} || exit_code=$?
          if [ "${exit_code:-0}" -eq 2 ]; then
            echo "Failing pipeline due to HIGH/CRITICAL vulnerabilities detected by Trivy."
            exit 1
          fi
        '''
      }
    }

    stage('Consolidar Reportes') {
      steps {
        echo "Consolidando todos los reportes de seguridad..."
        sh '''
          # Crear índice de reportes
          cat > ${REPORTS_DIR}/index.html << EOF
          <html>
          <head><title>Security Reports - Build ${BUILD_NUMBER}</title></head>
          <body>
          <h1>Security Reports - ${JOB_NAME} - Build ${BUILD_NUMBER}</h1>
          <ul>
          <li><a href="sast/semgrep-results.json">SAST - Semgrep Report (JSON)</a></li>
          <li><a href="sast/semgrep-results.sarif">SAST - Semgrep Report (SARIF)</a></li>
          <li><a href="sca/dependency-check-report.html">SCA - Dependency Check Report</a></li>
          <li><a href="container-scan/trivy-report.txt">Container Scan - Trivy Report</a></li>
          <li><a href="dast/zap-report.html">DAST - ZAP Report</a></li>
          </ul>
          <p>Generated on: $(date)</p>
          </body>
          </html>
          EOF
          
          # Crear resumen JSON
          cat > ${REPORTS_DIR}/summary.json << EOF
          {
            "buildNumber": "${BUILD_NUMBER}",
            "jobName": "${JOB_NAME}",
            "timestamp": "$(date -Iseconds)",
            "reports": {
              "sast": "security-reports/sast/semgrep-results.json",
              "sca": "security-reports/sca/dependency-check-report.json",
              "container": "security-reports/container-scan/trivy-report.json",
              "dast": "security-reports/dast/zap-report.json"
            }
          }
          EOF
          
          echo "=== ESTRUCTURA DE REPORTES ==="
          find ${REPORTS_DIR} -type f | sort
          echo "=== TAMAÑO DE REPORTES ==="
          du -sh ${REPORTS_DIR}/*
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'security-reports/**/*', allowEmptyArchive: true
          stash name: 'security-reports', includes: 'security-reports/**/*'
        }
      }
    }

  } // stages

  post {
    always {
      echo "=== PIPELINE COMPLETADO ==="
      echo "Reportes disponibles en: ${REPORTS_DIR}"
      echo "Build Number: ${BUILD_NUMBER}"
      
      script {
        // Publicar resultados de tests (si existen)
        junit allowEmptyResults: true, testResults: 'security-reports/tests/*.xml'
        
        // Notificación de resumen
        def reportFiles = findFiles(glob: 'security-reports/**/*')
        echo "Total de archivos de reporte generados: ${reportFiles.size()}"
      }
    }
    success {
      echo "✅ Pipeline ejecutado exitosamente"
      echo "📊 Reportes de seguridad disponibles en el workspace"
    }
    failure {
      echo "❌ Pipeline falló - Revisar reportes de seguridad"
    }
    unstable {
      echo "⚠️ Pipeline inestable - Posibles vulnerabilidades encontradas"
    }
  }
}