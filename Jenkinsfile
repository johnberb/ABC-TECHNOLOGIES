pipeline {
    agent any
  
    tools {
        maven 'maven3'  
    }
    
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        BUILD_NUMBER = "${env.BUILD_ID}"
        REMOTE_ARTIFACT_DIR = '/home/ansible/ansible/tmp/jenkins-artifacts'
        REPO_ROOT = "${WORKSPACE}"
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
                script {
                    // Verify POM location
                    sh 'ls -la ${WORKSPACE}/pom.xml || echo "POM not found in workspace root"'
                    
                    // Build with Maven
                    sh 'mvn -f ${WORKSPACE}/pom.xml clean package'
                    
                    // Archive test results and artifacts
                    junit '**/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
                }
            }
        }
        
        // STAGE 3: Prepare Deployment Files
        stage('Prepare Files') {
            steps {
                sh '''
                    # Create staging directory
                    mkdir -p ${WORKSPACE}/deployment-files
                    
                    # Copy all necessary files
                    cp ${WORKSPACE}/docker_build.yml ${WORKSPACE}/deployment-files/
                    cp ${WORKSPACE}/kube_deploy.yml ${WORKSPACE}/deployment-files/
                    cp ${WORKSPACE}/prometheus_deploy.yml ${WORKSPACE}/deployment-files/
                    cp ${WORKSPACE}/node-exporter.yaml ${WORKSPACE}/deployment-files/
                    
                    # Copy built WAR file
                    cp ${WORKSPACE}/target/ABCtechnologies-1.0.war ${WORKSPACE}/deployment-files/
                    
                    # Verify copied files
                    ls -la ${WORKSPACE}/deployment-files/
                '''
            }
        }
        
        // STAGE 4: Verify SSH Connection
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
        
        // STAGE 5: Transfer Files
        stage('Transfer Files') {
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
        
        // Remaining stages (5-8) remain unchanged from previous version
        // ...
    }
    
    post {
        always {
            cleanWs()
        }
    }
}