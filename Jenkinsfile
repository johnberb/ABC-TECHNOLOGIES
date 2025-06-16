pipeline {
    agent any
  
    tools {
        maven 'maven3'  
    }
    
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        BUILD_NUMBER = "${env.BUILD_ID}"
        REMOTE_ARTIFACT_DIR = '/home/ansible/ansible/tmp/jenkins-artifacts'
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
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' root@10.10.10.229 'whoami && pwd && mkdir -p ${REMOTE_ARTIFACT_DIR}'
                        """
                    }
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
                                    sourceFiles: '/var/lib/jenkins/jobs/java build/builds/12/archive/target/*.war',
                                    removePrefix: '/var/lib/jenkins/jobs/java build/builds/12/archive/target',
                                    remoteDirectory: REMOTE_ARTIFACT_DIR,
                                    execCommand: "chmod 644 ${REMOTE_ARTIFACT_DIR}/*.war"
                                )
                            ],
                            verbose: true
                        )
                    ]
                )
            }
        }
        
        // STAGE 5: Build Docker Image
        stage('Run Ansible Playbook') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        # Set strict permissions for the key
                        chmod 600 "$SSH_KEY"
        
                        # Verify the key works manually first
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 "echo 'SSH test successful'" || {
                            echo "ERROR: SSH test failed"
                            exit 1
                        }
        
                        # Run Ansible with explicit key and user
                        ansible-playbook \
                            -i /etc/ansible/hosts \
                            ${ANSIBLE_HOME}/playbooks/docker_build.yml \
                            --private-key="$SSH_KEY" \
                            --user=ansible@10.10.10.229 \
                            --extra-vars "artifact_path=/tmp/jenkins-artifacts/ABCtechnologies-1.0.war"
                    '''
                }
            }
        }
        
        // STAGE 6: Deploy to Kubernetes
        stage('Deploy to K8s') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' root@10.10.10.229 '
                                cd ${ANSIBLE_HOME} && \
                                ansible-playbook \
                                    -i /etc/ansible/hosts \
                                    playbooks/k8s_deploy.yml \
                                    --extra-vars \"image_tag=${BUILD_NUMBER}\"
                            '
                        """
                    }
                }
            }
        }
        
        // STAGE 7: Verify Deployment
        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        def result = sh(script: """
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' root@10.10.10.229 '
                                kubectl get pods -n default -l app=myapp && \
                                kubectl get svc myapp -n default
                            '
                        """, returnStdout: true)
                        echo "Deployment Status:\n${result}"
                    }
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
