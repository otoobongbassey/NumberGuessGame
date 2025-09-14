pipeline {
    agent any

    tools {
        jdk 'jdk8'             // Jenkins → Global Tool Configuration (Java 8 / 11 for build)
        maven 'maven-3.9'       // Jenkins → Global Tool Configuration
    }

    environment {
        SONARQUBE_ENV = 'MySonarQube'   // Name configured in Jenkins → Configure System
        NEXUS_REPO = 'releases'         // Nexus repository ID from pom.xml <distributionManagement>
        NEXUS_URL = 'http://54.165.30.243:8081/repository/releases/'
        DEPLOY_PATH = '/opt/tomcat/webapps'
        JAVA_8_HOME  = '/usr/lib/jvm/java-1.8.0-amazon-corretto.x86_64'
        JAVA_11_HOME = '/usr/lib/jvm/java-11-amazon-corretto.x86_64'
        
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/otoobongbassey/NumberGuessGame.git'
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                // Use Java 11 for SonarQube analysis
                withEnv(["JAVA_HOME=${env.JAVA_11_HOME}", "PATH=${env.JAVA_11_HOME}/bin:${env.PATH}"]) {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=NumberGuessGame'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Test') {
            steps {
                withEnv(["JAVA_HOME=${env.JAVA_11_HOME}", "PATH=${env.JAVA_11_HOME}/bin:${env.PATH}"]) {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                // If Nexus specifically requires Java 8, switch to it temporarily
                withEnv(["JAVA_HOME=${env.JAVA_8_HOME}", "PATH=${env.JAVA_8_HOME}/bin:${env.PATH}"]) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                sh '''
                WAR_FILE=$(ls target/*.war | head -n 1)
                if [ -f "$WAR_FILE" ]; then
                  echo "Deploying $WAR_FILE to Tomcat..."
                  sudo cp $WAR_FILE ${DEPLOY_PATH}/NumberGuessGame.war
                  sudo systemctl restart tomcat
                else
                  echo "WAR file not found! Build might have failed."
                  exit 1
                fi
                '''
            }
        }
    }

    post {
        success {
            emailext(
                to: 'otoobongbassey5@gmail.com',
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "✅ Build succeeded!\n\nCheck console: ${env.BUILD_URL}"
            )
        }
        failure {
            emailext(
                to: 'otoobongbassey@gmail.com',
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "❌ Build failed!\n\nCheck logs: ${env.BUILD_URL}"
            )
        }
    }
}
