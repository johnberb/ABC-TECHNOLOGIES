pipeline {
    agent any
    
    tools {
        maven 'maven3'  
    }
    
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        BUILD_NUMBER = "${env.BUILD_ID}"
    }
    
    stages {
        // STAGE 1: Code Checkout
        stage('Checkout') {
            steps {
                git branch: 'master', 
                     url: 'https://github.com/johnberb/ABC-TECHNOLOGIES.git'
            }
        }
        
        // STAGE 2: Build and Test
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
                junit '**/target/surefire-reports/*.xml'
                archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
            }
        }
        
        // STAGE 3: Verify SSH Connection
        stage('Test SSH Connection') {
            steps {
                script {
                    sshCommand(
                        remote: [
                            name: 'ansible',
                            host: '192.168.1.100',
                            user: 'ubuntu',
                            identity: 'jenkins-ssh-key'
                        ],
                        knownHosts: '10.10.10.229 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBF2mAO7g7oYm694zY8xzsVriAzwXW+VuThXQKZ5x+S1KPs283L2o4+esJdXcuDWDxFxpIGNtaTGNOLvWpsw6LaI=',
                        command: 'ls -la'
                    )
                }
            }
        }
        
        // STAGE 4: Transfer Artifacts
        stage('Transfer WAR File') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Ansible',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'target/*.war',
                                    removePrefix: 'target',
                                    remoteDirectory: '/tmp/jenkins-artifacts',
                                    execCommand: 'chmod 644 /tmp/jenkins-artifacts/*.war'
                                )
                            ],
                            verbose: true
                        )
                    ]
                )
            }
        }
        
        // STAGE 5: Build Docker Image
        stage('Build Docker Image') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Ansible',
                            transfers: [
                                sshTransfer(
                                    execCommand: """
                                    cd ${ANSIBLE_HOME} && \
                                    ansible-playbook \
                                        -i /etc/ansible/hosts.ini \
                                        playbooks/docker_build.yml \
                                        --extra-vars \"artifact_path=/tmp/jenkins-artifacts/ABCtechnologies-1.0.war\"
                                    """
                                )
                            ],
                            verbose: true
                        )
                    ]
                )
            }
        }
        
        // STAGE 6: Deploy to Kubernetes
        stage('Deploy to K8s') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Kubernetes',
                            transfers: [
                                sshTransfer(
                                    execCommand: """
                                    cd ${ANSIBLE_HOME} && \
                                    ansible-playbook \
                                        -i /etc/ansible/hosts.ini \
                                        playbooks/k8s_deploy.yml \
                                        --extra-vars \"image_tag=${BUILD_NUMBER}\"
                                    """
                                )
                            ],
                            verbose: true
                        )
                    ]
                )
            }
        }
        
        // STAGE 7: Verify Deployment
        stage('Verify Deployment') {
            steps {
                script {
                    def result = sshCommand(
                        remote: [name: 'Kubernetes'],
                        command: """
                        kubectl get pods -n default -l app=myapp && \
                        kubectl get svc myapp -n default
                        """
                    )
                    echo "Deployment Status:\n${result}"
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
    }
}
