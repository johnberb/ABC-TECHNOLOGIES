pipeline {
    agent any
    
    tools {
        maven 'maven3'  
    }
    
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        KUBECONFIG = '/home/docker/.kube/config'
        BUILD_NUMBER = "${env.BUILD_ID}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/johnberb/ABC-TECHNOLOGIES.git'
            }
        }
        
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
                junit '**/target/surefire-reports/*.xml'
                jacoco execPattern: '**/target/jacoco.exec',
                      classPattern: '**/target/classes',
                      sourcePattern: '**/src/main/java'
            }
        }
        
        stage('Prepare Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Ansible',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: '**/target/*.war',
                                    remoteDirectory: '/tmp/jenkins-artifacts/',
                                    execCommand: 'chmod 644 /tmp/jenkins-artifacts/*.war'
                                )
                            ]
                        )
                    ]
                )
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Ansible',
                            transfers: [
                                sshTransfer(
                                    execCommand: """
                                    ansible-playbook \
                                    -i /etc/ansible/hosts.ini \
                                    ${ANSIBLE_HOME}/playbooks/docker_build.yml \
                                    --extra-vars \"artifact_path=/tmp/jenkins-artifacts/ABCtechnologies-1.0.war\"
                                    """
                                )
                            ]
                        )
                    ]
                )
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Kubernetes',
                            transfers: [
                                sshTransfer(
                                    execCommand: """
                                    ansible-playbook \
                                    -i /etc/ansible/hosts.ini \
                                    ${ANSIBLE_HOME}/playbooks/k8s_deploy.yml \
                                    --extra-vars \"image_tag=${BUILD_NUMBER}\"
                                    """
                                )
                            ]
                        )
                    ]
                )
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verify = sshPublisher(
                        publishers: [
                            sshPublisherDesc(
                                configName: 'Kubernetes',
                                transfers: [
                                    sshTransfer(
                                        execCommand: """
                                        kubectl get pods -n default -l app=myapp && \
                                        kubectl get svc myapp -n default
                                        """
                                    )
                                ]
                            )
                        ]
                    )
                    
                    if (verify != 0) {
                        error('Deployment verification failed')
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(color: 'good', message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
        }
        failure {
            slackSend(color: 'danger', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
        }
    }
}
