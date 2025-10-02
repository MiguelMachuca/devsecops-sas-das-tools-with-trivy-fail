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
                        # Limpiar archivos previos
                        rm -f checkov-scan-results.*
                        
                        # Ejecutar Checkov - generará archivos en directorio results-checkov/
                        
                        checkov -f docker-compose.yml -f Dockerfile \
                          --soft-fail \
                          --output json --output-file-path checkov-results \
                          --output junitxml --output-file-path checkov-results
                                          
                        # Copiar y renombrar los archivos con nombres más descriptivos

                        cp checkov-results/results_json.json checkov-scan-results.json
                        cp checkov-results/results_junitxml.xml checkov-scan-results.xml
                        
                        # Limpiar archivos temporales y directorio
                      
                        rm -rf checkov-results/

                    '''
                }
            }
        }
        post {
            always {
                junit testResults: 'checkov-scan-results.xml', allowEmptyResults: true
                
                archiveArtifacts artifacts: 'checkov-scan-results.json, checkov-scan-results.xml', allowEmptyArchive: true
            }
        }
    }
  }

  post {
      always {
          echo "Pipeline execution completed - Status: ${currentBuild.result}"
          
          // Archivar TODOS los reportes de seguridad
          archiveArtifacts artifacts: '**/*report*, **/*results*, **/*.xml, **/*.json', allowEmptyArchive: true
          
          // Publicar reportes consolidados
          junit testResults: '**/*.xml', allowEmptyResults: true
          dependencyCheckPublisher pattern: 'dependency-check-report.xml'
          
          // Métricas y estadísticas
          script {
              echo "Build Number: ${env.BUILD_NUMBER}"
              echo "Build URL: ${env.BUILD_URL}"
              echo "Duration: ${currentBuild.durationString}"
          }
      }
      
      success {
          echo "✅ Pipeline ejecutado EXITOSAMENTE"
          script {
              // Notificación de éxito
              emailext (
                  subject: "✅ PIPELINE SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                  body: """
                  Pipeline completado exitosamente:
                  
                  - Build: ${env.BUILD_URL}
                  - Duración: ${currentBuild.durationString}
                  - Commit: ${env.GIT_COMMIT ?: 'N/A'}
                  
                  Reportes disponibles en los artifacts del build.
                  """,
                  to: "mmangeliguel@gmail.com"
              )
          }
      }
      
      failure {
          echo "❌ Pipeline FALLÓ"
          script {
              // Notificación de fallo con detalles
              emailext (
                  subject: "🚨 PIPELINE FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                  body: """
                  El pipeline ha fallado:
                  
                  - Build: ${env.BUILD_URL}
                  - Stage que falló: ${env.STAGE_NAME}
                  - Duración: ${currentBuild.durationString}
                  
                  Por favor revisar los logs para más detalles.
                  """,
                  to: "mmangeliguel@gmail.com"
              )
          }
      }
      
      unstable {
          echo "⚠️  Pipeline marcado como INESTABLE - Vulnerabilidades HIGH/CRITICAL detectadas"
          script {
              // Notificación específica para vulnerabilidades
              emailext (
                  subject: "⚠️  PIPELINE UNSTABLE: Vulnerabilidades en ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                  body: """
                  Pipeline completado pero con vulnerabilidades CRITICAL/HIGH:
                  
                  - Build: ${env.BUILD_URL}
                  - Razón: Vulnerabilidades detectadas por Trivy/Policy Check
                  - Acción: Revisar reportes de seguridad
                  
                  Se requiere revisión manual.
                  """,
                  to: "mmangeliguel@gmail.com"
              )
          }
      }
      
      changed {
          echo "📊 Estado del pipeline cambió respecto a la última ejecución"
          script {
              if (currentBuild.previousBuild) {
                  echo "Estado anterior: ${currentBuild.previousBuild.result}"
                  echo "Estado actual: ${currentBuild.result}"
              }
          }
      }
      
      cleanup {
          echo "🧹 Ejecutando limpieza final..."
          // Limpieza garantizada de recursos
          sh '''
              # Limpiar contenedores detenidos
              docker-compose -f docker-compose.yml down || true
              
              # Limpiar imágenes temporales
              docker image prune -f || true
              
              # Limpiar redes no utilizadas
              docker network prune -f || true
          '''
      }
  }
}