pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-amazon-corretto.x86_64'
        PATH = "${JAVA_HOME}/bin:${PATH}"
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
        withSonarQubeEnv('sonar-server') {
            withCredentials([
                string(
                    credentialsId: 'sonar-token',
                    variable: 'SONAR_TOKEN'
                )
            ]) {
                sh '''
                    mvn -e -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                      -Dsonar.projectKey=DevOps-Calculator-Web-Ponpandian \
                      -Dsonar.projectName="DevOps Calculator Web - Ponpandian" \
                      -Dsonar.host.url="$SONAR_HOST_URL" \
                      -Dsonar.token="$SONAR_TOKEN" \
                      -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                      -Dsonar.coverage.exclusions=**/CalculatorServlet.java
                '''
            }
        }
    }
}

    

     stage('Verify WAR') {
            steps {
                sh '''
                    test -f target/calculator.war
                    ls -lh target/calculator.war
                '''
            }
        }

            stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}

        stage('Deploy to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-Ponpandian',
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
                    '''
                }
            }
        }
        stage('Provision Tomcat Server via Ansible') {
    steps {
        withCredentials([
            sshUserPrivateKey(
                credentialsId: 'ansible-target-ssh-student',
                keyFileVariable: 'ANSIBLE_SSH_KEY',
                usernameVariable: 'ANSIBLE_SSH_USER'
            )
        ]) {
            sh '''
                set -eu
                export ANSIBLE_CONFIG="$WORKSPACE/ansible/ansible.cfg"
                INVENTORY="$WORKSPACE/ansible/inventory.ini"

                ansible calculator -i "$INVENTORY" -m ping \
                    --private-key="$ANSIBLE_SSH_KEY" \
                    -e "ansible_user=$ANSIBLE_SSH_USER"

                ansible-playbook -i "$INVENTORY" \
                    "$WORKSPACE/ansible/install-java.yml" \
                    --private-key="$ANSIBLE_SSH_KEY" \
                    -e "ansible_user=$ANSIBLE_SSH_USER"

                ansible-playbook -i "$INVENTORY" \
                    "$WORKSPACE/ansible/install-tomcat.yml" \
                    --private-key="$ANSIBLE_SSH_KEY" \
                    -e "ansible_user=$ANSIBLE_SSH_USER"
            '''
        }
    }
}
    }

    stage('Deploy to Tomcat via Ansible') {
    steps {
        withCredentials([
            sshUserPrivateKey(
                credentialsId: 'ansible-target-ssh-student',
                keyFileVariable: 'ANSIBLE_SSH_KEY',
                usernameVariable: 'ANSIBLE_SSH_USER'
            ),
            string(
                credentialsId: 'ansible-vault-password',
                variable: 'ANSIBLE_VAULT_PASSWORD'
            )
        ]) {
            sh '''
                set +x
                set -eu
                export ANSIBLE_CONFIG="$WORKSPACE/ansible/ansible.cfg"
                INVENTORY="$WORKSPACE/ansible/inventory.ini"
                PLAYBOOK="$WORKSPACE/ansible/deploy-calculator.yml"

                APP_VERSION="$(mvn -q -DforceStdout \
                    help:evaluate -Dexpression=project.version)"
                test -n "$APP_VERSION"
                test -r "$ANSIBLE_SSH_KEY"
                ssh-keygen -y -f "$ANSIBLE_SSH_KEY" >/dev/null

                ansible calculator -i "$INVENTORY" -m ping \
                    --private-key="$ANSIBLE_SSH_KEY" \
                    -e "ansible_user=$ANSIBLE_SSH_USER"

                VAULT_PASS_FILE="$(mktemp)"
                trap 'rm -f "$VAULT_PASS_FILE"' EXIT
                chmod 600 "$VAULT_PASS_FILE"
                printf '%s' "$ANSIBLE_VAULT_PASSWORD" > "$VAULT_PASS_FILE"
                unset ANSIBLE_VAULT_PASSWORD

                ansible-playbook -i "$INVENTORY" "$PLAYBOOK" \
                    --vault-password-file "$VAULT_PASS_FILE" \
                    --private-key="$ANSIBLE_SSH_KEY" \
                    -e "ansible_user=$ANSIBLE_SSH_USER" \
                    -e "artifact_version=$APP_VERSION"
            '''
        }
    }
}


    post {
        always {
            junit allowEmptyResults: true,
                  testResults: '*/target/surefire-reports/.xml'

            archiveArtifacts artifacts: 'target/*.war',
                             fingerprint: true
        }

        success {
            echo 'Build, JUnit tests, WAR verification, and artifact archiving succeeded.'
        }

        failure {
            echo 'Pipeline failed. Review the Jenkins console output.'
        }
    }
}