pipeline {
  agent any

  tools { maven 'maven3' }

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Desplegar a Staging y hacer health check')
    string(name: 'STAGING_HOST', defaultValue: 'host.docker.internal', description: 'Host/IP del Staging (Docker Desktop)')
    string(name: 'STAGING_USER', defaultValue: 'staging', description: 'Usuario SSH en Staging')
    string(name: 'STAGING_PORT', defaultValue: '2222', description: 'Puerto SSH del Staging')
    string(name: 'STAGING_HTTP_PORT', defaultValue: '8081', description: 'Puerto HTTP publicado en el host')
  }

  environment {
    SSH_CRED_ID  = 'staging-ssh'
    STAGING_DIR  = '/opt/springapp'
    SERVICE_NAME = 'springapp'
    HEALTH_PATH  = '/'   // si luego agregas actuator, cambia a '/actuator/health'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Unit Tests') {
      steps { sh 'mvn -B -U -e -DskipTests=false clean test' }
      post { always { junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true } }
    }

    stage('Code Quality (Checkstyle)') {
      steps { sh 'mvn -B checkstyle:check' }
    }

    stage('Coverage (JaCoCo)') {
      steps { sh 'mvn -B -DskipTests verify' }
    }

    stage('Package (.jar)') {
      steps { sh 'mvn -B -DskipTests package' }
    }

    stage('Deploy to Staging (SSH)') {
      when {
        expression { return params.DEPLOY && params.STAGING_HOST?.trim() }
      }
      steps {
        sshagent(credentials: [env.SSH_CRED_ID]) {
          sh """
            set -eux
            JAR=\$(ls target/*.jar | head -n 1)

            # Crear carpeta destino si no existe
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} "mkdir -p ${STAGING_DIR}"

            # Copiar el artefacto
            scp -P ${STAGING_PORT} -o StrictHostKeyChecking=no "\$JAR" ${STAGING_USER}@${STAGING_HOST}:${STAGING_DIR}/${SERVICE_NAME}.jar

            # Reiniciar proceso (si hay) y arrancar en 8080 dentro del contenedor
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} '
              set -eu
              (pgrep -f "${STAGING_DIR}/${SERVICE_NAME}.jar" && pkill -f "${STAGING_DIR}/${SERVICE_NAME}.jar" || true)
              nohup java -jar ${STAGING_DIR}/${SERVICE_NAME}.jar --server.port=8080 > ${STAGING_DIR}/${SERVICE_NAME}.log 2>&1 &
              sleep 2
            '
          """
        }
      }
    }

    stage('Validate Deployment (Health)') {
      when {
        expression { return params.DEPLOY && params.STAGING_HOST?.trim() }
      }
      steps {
        sh """
          set -eux
          for i in \$(seq 1 30); do
            CODE=\$(curl -s -o /dev/null -w "%{http_code}" "http://${STAGING_HOST}:${STAGING_HTTP_PORT}${HEALTH_PATH}" || true)
            echo "HTTP=\$CODE"
            [ "\$CODE" = "200" ] && exit 0
            sleep 2
          done
          echo "Health check FAILED"
          exit 1
        """
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'target/*.jar, target/site/jacoco/**/*, target/checkstyle-result.xml',
                       allowEmptyArchive: true, fingerprint: true
      junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
    }
    success { echo 'Pipeline OK.' }
  }
}
