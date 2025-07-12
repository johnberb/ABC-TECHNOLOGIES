pipeline {
    agent any
  
    tools {
        maven 'maven3'  
    }
    
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        BUILD_NUMBER = "${env.BUILD_ID}"
        REMOTE_ARTIFACT_DIR = '/home/ansible/ansible/tmp/jenkins-artifacts'
        REPO_ROOT = "${WORKSPACE}/ABC-TECHNOLOGIES"
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
                dir('ABC-TECHNOLOGIES') {
                    sh 'mvn clean package'
                    junit '**/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
                }
            }
        }
        
        // NEW STAGE: Prepare Files from Repository Root
        stage('Prepare Deployment Files') {
            steps {
                sh '''
                    # Verify root-level files
                    ls -la ${REPO_ROOT}/
                    
                    # Create staging directory
                    mkdir -p ${WORKSPACE}/deployment-files
                    
                    # Copy all necessary files from repository root
                    cp ${REPO_ROOT}/docker_build.yml ${WORKSPACE}/deployment-files/
                    cp ${REPO_ROOT}/kube_deploy.yml ${WORKSPACE}/deployment-files/
                    cp ${REPO_ROOT}/prometheus_deploy.yml ${WORKSPACE}/deployment-files/
                    cp ${REPO_ROOT}/node-exporter.yaml ${WORKSPACE}/deployment-files/
                    
                    # Copy WAR file from target
                    cp ${REPO_ROOT}/target/ABCtechnologies-1.0.war ${WORKSPACE}/deployment-files/
                    
                    # Verify copied files
                    ls -la ${WORKSPACE}/deployment-files/
                '''
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
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 'mkdir -p /home/ansible/ansible/playbooks'
                        """
                    }
                }
            }
        }
        
        // STAGE 4: Transfer All Files
        stage('Transfer Deployment Files') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            # Transfer WAR file
                            scp -i "$SSH_KEY" \
                                "${WORKSPACE}/deployment-files/ABCtechnologies-1.0.war" \
                                ansible@10.10.10.229:"${REMOTE_ARTIFACT_DIR}/"
                                
                            # Transfer playbooks
                            scp -i "$SSH_KEY" \
                                "${WORKSPACE}/deployment-files/docker_build.yml" \
                                ansible@10.10.10.229:"/home/ansible/ansible/playbooks/"
                                
                            scp -i "$SSH_KEY" \
                                "${WORKSPACE}/deployment-files/kube_deploy.yml" \
                                ansible@10.10.10.229:"/home/ansible/ansible/playbooks/"
                                
                            scp -i "$SSH_KEY" \
                                "${WORKSPACE}/deployment-files/prometheus_deploy.yml" \
                                ansible@10.10.10.229:"/home/ansible/ansible/playbooks/"
                                
                            # Transfer monitoring file
                            scp -i "$SSH_KEY" \
                                "${WORKSPACE}/deployment-files/node-exporter.yaml" \
                                ansible@10.10.10.229:"/tmp/"
                        """
                    }
                }
            }
        }
        
        // STAGE 5: Build Docker Image
        stage('Run Docker Build') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        # Copy private key temporarily
                        scp -i "$SSH_KEY" "$SSH_KEY" ansible@10.10.10.229:/home/ansible/.ssh/jenkins_key
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 "chmod 600 ~/.ssh/jenkins_key"
                        
                        # Run Docker build playbook
                        ssh -i "$SSH_KEY" ansible@10.10.10.229 "
                            cd /home/ansible/ansible &&
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/docker_build.yml \
                                --extra-vars 'artifact_path=${REMOTE_ARTIFACT_DIR}/ABCtechnologies-1.0.war'
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
        
        // STAGE 8: Deploy Monitoring
        stage('Deploy Monitoring') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
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
    }
    
    post {
        always {
            cleanWs()
        }
    }
}