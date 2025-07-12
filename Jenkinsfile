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
                        # 1. Copy WAR from Jenkins to Ansible server
                        scp -i "$SSH_KEY" \
                            "/var/lib/jenkins/jobs/java build/builds/1/archive/target/ABCtechnologies-1.0.war" \
                            ansible@10.10.10.229:"/home/ansible/ansible/tmp/jenkins-artifacts/"

                        # 2. Verify file transfer
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 \
                            "ls -l /home/ansible/ansible/tmp/jenkins-artifacts/ABCtechnologies-1.0.war"
                    
                        #copy private key temporarily onto the ansible server
                        scp -i "$SSH_KEY" "$SSH_KEY" ansible@10.10.10.229:/home/ansible/.ssh/jenkins_key
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 "chmod 600 ~/.ssh/jenkins_key"
  
                        # Test SSH connection first
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 "echo 'SSH test successful'"
        
                        # Run Ansible PLAYBOOK on the Ansible server (not locally)
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 "
                            # Download and overwrite docker_build.yml from GitHub
                            curl -o /home/ansible/ansible/playbooks/docker_build.yml \
                                https://raw.githubusercontent.com/johnberb/ABC-TECHNOLOGIES/master/ansible/playbooks/docker_build.yml
                            
                            # Verify file was downloaded
                            ls -l /home/ansible/ansible/playbooks/docker_build.yml
                            
                            # Run the playbook
                            cd /home/ansible/ansible &&
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/docker_build.yml \
                                --extra-vars 'artifact_path=/tmp/jenkins-artifacts/ABCtechnologies-1.0.war'
                        "
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
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                                cd ${ANSIBLE_HOME} && \
                                ansible-playbook \
                                    -i /etc/ansible/hosts \
                                    playbooks/kube_deploy.yml \
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
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get pods -n default -l app=myapp && \
                                kubectl get svc myapp -n default
                            '
                        """, returnStdout: true)
                        echo "Deployment Status:\n${result}"
                    }
                }
            }
        }
        
        // STAGE 8: Deploy Node Exporter
        stage('Deploy Monitoring') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            scp -i '$SSH_KEY' monitoring/node-exporter.yaml ansible@10.10.10.229:/tmp/
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl create namespace monitoring || true
                                kubectl apply -f /tmp/node-exporter.yaml
                                kubectl rollout status daemonset/node-exporter -n monitoring --timeout=120s
                            '
                        """
                    }
                }
            }
        }
        
        // STAGE 9: Deploy Prometheus and Grafana
        stage('Deploy Observability Stack') {
            steps {
              script {
                    withCredentials([sshUserPrivateKey(       
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                                cd ${ANSIBLE_HOME} && \
                                ansible-playbook \
                                    -i /etc/ansible/hosts \
                                    playbooks/prometheus_deploy.yml \
                                    --extra-vars \"image_tag=${BUILD_NUMBER}\"
                            '
                        """
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