pipeline {
  agent any

  tools {
    maven 'maven3'       // Manage Jenkins → Tools
  }

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Desplegar a Staging y hacer health check')
    string(name: 'STAGING_HOST', defaultValue: 'REEMPLAZA-CON-IP-O-DNS', description: 'Host/IP del Staging')
    string(name: 'STAGING_USER', defaultValue: 'staging', description: 'Usuario SSH en Staging')
  }

  environment {
    SSH_CRED_ID  = 'staging-ssh'    // Credencial SSH en Jenkins
    STAGING_DIR  = '/opt/springapp'
    SERVICE_NAME = 'springapp'
    HEALTH_PATH  = '/health'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm   // porque el job será "Pipeline from SCM"
      }
    }

    stage('Build & Unit Tests') {
      steps {
        sh 'mvn -B -U -e -DskipTests=false clean test'
      }
      post {
        always {
          junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
        }
      }
    }

    stage('Code Quality (Checkstyle)') {
      steps {
        sh 'mvn -B checkstyle:check'
      }
    }

    stage('Coverage (JaCoCo)') {
      steps {
        // 'verify' ejecuta jacoco:check con las reglas del POM
        // (añadimos -DskipTests para no volver a correr tests)
        sh 'mvn -B -DskipTests verify'
      }
    }

    stage('Package (.jar)') {
      steps {
        sh 'mvn -B -DskipTests package'
      }
    }

    stage('Deploy to Staging (SSH)') {
      when {
        allOf {
          branch 'main'
          expression { return params.DEPLOY && params.STAGING_HOST?.trim() }
        }
      }
      steps {
        sshagent(credentials: [env.SSH_CRED_ID]) {
          sh '''
            set -euxo pipefail
            JAR=$(ls target/*.jar | head -n 1)
            ssh -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} "mkdir -p ${STAGING_DIR}"
            scp -o StrictHostKeyChecking=no "$JAR" ${STAGING_USER}@${STAGING_HOST}:${STAGING_DIR}/${SERVICE_NAME}.jar

            # reinicia servicio si existe; si no, levanta con nohup
            ssh -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} "\
              (sudo systemctl daemon-reload || true) && \
              (sudo systemctl restart ${SERVICE_NAME}.service || \
               (nohup java -jar ${STAGING_DIR}/${SERVICE_NAME}.jar >/var/log/${SERVICE_NAME}.log 2>&1 &))"
          '''
        }
      }
    }

    stage('Validate Deployment (Health)') {
      when {
        allOf {
          branch 'main'
          expression { return params.DEPLOY && params.STAGING_HOST?.trim() }
        }
      }
      steps {
        sh '''
          set -euxo pipefail
          for i in $(seq 1 30); do
            OUT=$(curl -fsS "http://${STAGING_HOST}:8080${HEALTH_PATH}" || true)
            echo "$OUT"
            echo "$OUT" | grep -Eqi 'UP|200|ok' && exit 0
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
    success { echo 'Pipeline OK.' }
  }
}
