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
    string(name: 'STAGING_HOST', defaultValue: 'host.docker.internal', description: 'Host/IP del Staging')
    string(name: 'STAGING_USER', defaultValue: 'staging', description: 'Usuario SSH en Staging')
    string(name: 'STAGING_PORT', defaultValue: '2222', description: 'Puerto SSH del Staging')
  }

  environment {
    SSH_CRED_ID  = 'staging-ssh'    // Credencial SSH en Jenkins (clave privada)
    STAGING_DIR  = '/opt/springapp'
    SERVICE_NAME = 'springapp'
    HEALTH_PATH  = '/health'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm   // porque el job es "Pipeline from SCM"
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

            # Crear carpeta destino
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
              ${STAGING_USER}@${STAGING_HOST} "mkdir -p ${STAGING_DIR}"

            # Copiar artefacto
            scp -P ${STAGING_PORT} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
              "$JAR" ${STAGING_USER}@${STAGING_HOST}:${STAGING_DIR}/${SERVICE_NAME}.jar

            # Arrancar app (sin systemd; con nohup) en 8080 dentro del host
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
              ${STAGING_USER}@${STAGING_HOST} "\
                pkill -f '${STAGING_DIR}/${SERVICE_NAME}.jar' || true; \
                nohup java -jar ${STAGING_DIR}/${SERVICE_NAME}.jar --server.port=8080 \
                  > ${STAGING_DIR}/${SERVICE_NAME}.log 2>&1 & \
                sleep 2; \
                pgrep -fl java || true"
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
        // Validamos DESDE el host remoto (la app escucha en 8080 dentro)
        sshagent(credentials: [env.SSH_CRED_ID]) {
          sh '''
            set -euxo pipefail
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
              ${STAGING_USER}@${STAGING_HOST} '\
                for i in $(seq 1 30); do
                  OUT=$(curl -fsS http://localhost:8080'${HEALTH_PATH}' || true)
                  echo "$OUT"
                  echo "$OUT" | grep -Eqi "UP|200|ok|OK" && exit 0
                  sleep 2
                done
                echo "Health check FAILED"; \
                echo "---- Últimas líneas del log ----"; \
                tail -n 200 '${STAGING_DIR}'/'${SERVICE_NAME}'.log; \
                exit 1'
          '''
        }
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
