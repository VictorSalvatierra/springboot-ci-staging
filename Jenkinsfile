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
    string(name: 'STAGING_HOST', defaultValue: 'spring-staging', description: 'Host/IP del Staging dentro de la red Docker (ci-net)')
    string(name: 'STAGING_USER', defaultValue: 'staging', description: 'Usuario SSH en Staging')
    string(name: 'STAGING_PORT', defaultValue: '22', description: 'Puerto SSH del Staging (22 en la red interna)')
    string(name: 'STAGING_HTTP_PORT', defaultValue: '8080', description: 'Puerto HTTP interno expuesto por la app (red Docker)')
  }

  environment {
    SSH_CRED_ID  = 'staging-ssh'
    STAGING_DIR  = '/opt/springapp'
    SERVICE_NAME = 'springapp'
    HEALTH_PATH  = '/health'
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
      when { expression { return params.DEPLOY && params.STAGING_HOST?.trim() } }
      steps {
        sshagent(credentials: [env.SSH_CRED_ID]) {
          // 1) Crear carpeta y copiar el jar
          sh '''
            set -eux
            JAR=$(ls target/*.jar | head -n 1)
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} "mkdir -p ${STAGING_DIR}"
            scp -P ${STAGING_PORT} -o StrictHostKeyChecking=no "$JAR" ${STAGING_USER}@${STAGING_HOST}:${STAGING_DIR}/${SERVICE_NAME}.jar
          '''

          // 2) Reiniciar/arrancar proceso (no fallar la etapa por código de salida ssh)
          sh(returnStatus: true, script: '''
            set -eux
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} '
              set -e
              pkill -f "${STAGING_DIR}/${SERVICE_NAME}.jar" || true
              nohup java -jar ${STAGING_DIR}/${SERVICE_NAME}.jar --server.port=8080 > ${STAGING_DIR}/${SERVICE_NAME}.log 2>&1 &
              sleep 3
              exit 0
            '
          ''')
        }
      }
    }

    stage('Validate Deployment (Health)') {
      when { expression { return params.DEPLOY && params.STAGING_HOST?.trim() } }
      steps {
        sh '''
          set -eux
          for i in $(seq 1 30); do
            if OUT=$(curl -fsS "http://${STAGING_HOST}:${STAGING_HTTP_PORT}${HEALTH_PATH}"); then
              echo "$OUT"
              exit 0
            fi
            sleep 2
          done
          echo "Health check FAILED"
          exit 1
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'target/*.jar, target/site/jacoco/**/*, target/checkstyle-result.xml', allowEmptyArchive: true, fingerprint: true
      junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
    }
    failure {
      script {
        if (params.DEPLOY && params.STAGING_HOST?.trim()) {
          sshagent(credentials: [env.SSH_CRED_ID]) {
            sh '''
              set +e
              ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} \
                "echo '---- last app log ----'; tail -n 200 ${STAGING_DIR}/${SERVICE_NAME}.log || true"
            '''
          }
        }
      }
    }
    success { echo 'Pipeline OK.' }
  }
}
