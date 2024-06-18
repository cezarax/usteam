pipeline { 
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-repo')
    }
    stages {
        stage('Code analisys stage') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Quality gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true 
                }
            }
        }
        stage('Build artefacts') {
            steps {
                sh 'mvn clean install -DskipTests -Dcheckstyle.skip'
            }
        }
        stage('OWASP Dependency Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build Docker image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/myapp .'
            }
        }
        stage('Push artifacts to nexus-repo') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
                classifier: '',
                file: 'target/spring-petclinic-2.4.2.war',
                type: 'war']],
                credentialsId: 'nexus-creds',
                groupId: 'Petclinic',
                nexusUrl: 'nexus.traveldeals.tech',
                nexusVersion: 'nexus3',
                protocol: 'https',
                repository: 'nexus-repo',
                version: '1.0'
            }
        }
        stage('Trivy file scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage('Login to Nexus repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Push image to Nexus repo') {
            steps {
                sh 'docker push $NEXUS_REPO/myapp'
            }
        }
        stage('Trivi image scan') {
            steps {
                sh "trivy image $NEXUS_REPO/myapp > trivy.txt"
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@10.0.3.49 -o strictHostKeyChecking=no "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"'
                }
            }
        }
        stage('check stage website availability') {
            steps {
                 sh "sleep 40"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://stage.traveldeals.tech"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.traveldeals.tech", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic website is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic wordpress website appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@10.0.3.49 -o strictHostKeyChecking=no "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"'
                }
            }
        }
        stage('check prod website availability') {
            steps {
                 sh "sleep 40"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://prod.traveldeals.tech"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.traveldeals.tech", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The prod petclinic website is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The prod petclinic wordpress website appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
