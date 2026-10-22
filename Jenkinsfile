
pipeline {
    agent any

    tools {
        jdk 'Java21'
        maven 'Maven3'
    }

    environment {
        SONAR_HOST_URL = 'http://sonar.leancloudandevops.online:9000'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn -B clean verify'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml',
                          allowEmptyResults: true

                    archiveArtifacts artifacts: 'target/*.war',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Verify JaCoCo Report') {
            steps {
                sh '''
                    test -s target/site/jacoco/jacoco.xml
                    test -s target/site/jacoco/index.html
                    echo "JaCoCo XML and HTML reports found."
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {
                    sh '''
                        mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.projectKey=DevOps-Calculator-Web-Ponpandian \
                          -Dsonar.projectName="DevOps Calculator Web - Ponpandian" \
                          -Dsonar.host.url="$SONAR_HOST_URL" \
                          -Dsonar.token="$SONAR_TOKEN" \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        stage('Verify WAR') {
            steps {
                sh '''
                    test -s target/calculator.war
                    ls -lh target/calculator.war
                '''
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-ponpandian',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        umask 077

                        cat > "$WORKSPACE/.mvn-settings.xml" <<EOF
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0">
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${NEXUS_USERNAME}</username>
      <password>${NEXUS_PASSWORD}</password>
    </server>
  </servers>
</settings>
EOF

                        mvn -B -s "$WORKSPACE/.mvn-settings.xml" deploy

                        rm -f "$WORKSPACE/.mvn-settings.xml"
                    '''
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: '**/target/surefire-reports/*.xml'

            archiveArtifacts artifacts: 'target/*.war',
                             fingerprint: true,
                             allowEmptyArchive: true
        }

        success {
            echo 'Build, tests, SonarQube analysis, WAR verification, and deployment succeeded.'
        }

        failure {
            echo 'Pipeline failed. Review the Jenkins console output.'
        }

        cleanup {
            sh 'rm -f "$WORKSPACE/.mvn-settings.xml"'
        }
    }
}
