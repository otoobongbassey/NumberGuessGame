pipeline {
    agent any

    tools {
        jdk 'jdk11'
        maven 'maven-3.9'
    }

    environment {
        SONARQUBE_ENV = 'MySonarQube'
        NEXUS_REPO = 'releases'
        NEXUS_URL = 'http://34.235.118.123:8081/repository/releases/'
        DEPLOY_PATH = '/opt/tomcat/webapps'
        BACKUP_PATH = '/opt/tomcat/backup'
        JAVA_8_HOME  = '/usr/lib/jvm/java-1.8.0-amazon-corretto.x86_64'
        JAVA_11_HOME = '/usr/lib/jvm/java-11-amazon-corretto.x86_64'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                git branch: 'main', url: 'https://github.com/otoobongbassey/NumberGuessGame.git'
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                withSonarQubeEnv('MySonarQube') {
                    withEnv(["JAVA_HOME=${env.JAVA_11_HOME}", "PATH=${env.JAVA_11_HOME}/bin:${env.PATH}"]) {
                        sh "mvn verify sonar:sonar -Dsonar.projectKey=NumberGuessGame"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Test') {
            steps {
                sh 'sudo systemctl stop tomcat || true'
                withEnv(["JAVA_HOME=${env.JAVA_11_HOME}", "PATH=${env.JAVA_11_HOME}/bin:${env.PATH}"]) {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    withEnv([
                        "JAVA_HOME=${env.JAVA_11_HOME}",
                        "PATH=${env.JAVA_11_HOME}/bin:${env.PATH}"
                    ]) {
                        sh """
                          mkdir -p \$WORKSPACE/.m2
                          cat > \$WORKSPACE/.m2/settings.xml <<EOF
                          <settings>
                            <servers>
                              <server>
                                <id>${env.NEXUS_REPO}</id>
                                <username>\$NEXUS_USER</username>
                                <password>\$NEXUS_PASS</password>
                              </server>
                            </servers>
                          </settings>
                          EOF

                          mvn deploy --settings \$WORKSPACE/.m2/settings.xml
                        """
                    }
                }
            }
        }

        stage('Deploy to Tomcat with Rollback') {
            steps {
                sh '''
                set -e
                WAR_FILE=$(ls target/*.war | head -n 1)

                if [ -f "$WAR_FILE" ]; then
                  echo "Preparing backup directory..."
                  sudo mkdir -p ${BACKUP_PATH}

                  if [ -f "${DEPLOY_PATH}/NumberGuessGame.war" ]; then
                    echo "Backing up current WAR..."
                    sudo cp ${DEPLOY_PATH}/NumberGuessGame.war ${BACKUP_PATH}/NumberGuessGame.war.bak
                  fi

                  echo "Cleaning old deployment..."
                  sudo rm -rf ${DEPLOY_PATH}/NumberGuessGame ${DEPLOY_PATH}/NumberGuessGame.war

                  echo "Deploying new WAR..."
                  sudo cp $WAR_FILE ${DEPLOY_PATH}/NumberGuessGame.war
                  sudo systemctl restart tomcat

                  echo "Waiting for Tomcat to come up..."
                  sleep 10

                  echo "Performing health check..."
                  if curl -s --head http://localhost:8080/NumberGuessGame | grep "200 OK" > /dev/null; then
                    echo "✅ Deployment successful!"
                  else
                    echo "❌ Deployment failed, rolling back..."
                    if [ -f "${BACKUP_PATH}/NumberGuessGame.war.bak" ]; then
                      sudo cp ${BACKUP_PATH}/NumberGuessGame.war.bak ${DEPLOY_PATH}/NumberGuessGame.war
                      sudo systemctl restart tomcat
                      echo "Rollback completed."
                    else
                      echo "No backup WAR found, cannot rollback!"
                      exit 1
                    fi
                  fi
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
                to: 'basseyotoobong5@gmail.com',
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>✅ Good news!</p>
                         <p>The Jenkins job <b>${env.JOB_NAME} [${env.BUILD_NUMBER}]</b> completed successfully.</p>
                         <p>Check the <a href="${env.BUILD_URL}">build logs</a> for details.</p>"""
            )
        }
        failure {
            emailext(
                to: 'basseyotoobong5@gmail.com',
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>❌ Unfortunately, the Jenkins job <b>${env.JOB_NAME} [${env.BUILD_NUMBER}]</b> has failed.</p>
                         <p>Check the <a href="${env.BUILD_URL}">build logs</a> for details.</p>"""
            )
        }
    }
}
