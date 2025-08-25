pipeline {
    agent any
    environment {
        STAGING_SERVER = 'user@your-staging-server'
        ARTIFACT_NAME = 'demo-0.0.1-SNAPSHOT.jar'
    }
    stages {
        stage('Clone Repository') {
            steps {
                git 'https://github.com/your-org/your-springboot-repo.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Code Quality') {
            steps {
                sh 'mvn checkstyle:check'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Code Coverage') {
            steps {
                sh 'mvn jacoco:report'
            }
        }
        stage('Deploy to Staging') {
            steps {
                sh 'scp target/${ARTIFACT_NAME} $STAGING_SERVER:/home/your-user/staging/'
                sh 'ssh $STAGING_SERVER "nohup java -jar /home/your-user/staging/${ARTIFACT_NAME} > /dev/null 2>&1 &"'
            }
        }
        stage('Validate Deployment') {
            steps {
                sh 'sleep 10'
                sh 'curl --fail http://your-staging-server:8080/health'
            }
        }
    }
}
