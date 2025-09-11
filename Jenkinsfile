pipeline {
    agent any

    tools {
        jdk 'jdk17'             // Jenkins → Global Tool Configuration
        maven 'maven-3.9'       // Jenkins → Global Tool Configuration
    }

    environment {
        SONARQUBE_ENV = 'MySonarQube'   // Configure in Jenkins → Manage Jenkins → Configure System
        NEXUS_REPO = 'releases'         // Nexus repository ID from pom.xml <distributionManagement>
        NEXUS_URL = 'http://54.211.175.36:8081/repository/releases/'
        DEPLOY_PATH = '/opt/tomcat/webapps'
    }

    triggers {
        githubPush()   // Webhook trigger from GitHub → Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/otoobongbassey/NumberGuessGame.git'
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                sh 'mvn deploy'
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
