pipeline {
    agent any
  
    tools {
        maven 'maven3'  
    }
    
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        BUILD_NUMBER = "${env.BUILD_ID}"
        REMOTE_ARTIFACT_DIR = '/home/ansible/ansible/tmp/jenkins-artifacts'
        PROMETHEUS_VERSION = 'v2.47.2'
        GRAFANA_VERSION = '10.2.3'
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
                            ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                                whoami && \
                                pwd && \
                                mkdir -p ${REMOTE_ARTIFACT_DIR} && \
                                mkdir -p ${ANSIBLE_HOME}/playbooks
                            '
                        """
                    }
                }
            }
        }
        
        // STAGE 4: Transfer Artifacts
        stage('Transfer Files') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        // Transfer WAR file
                        sh """
                            scp -o StrictHostKeyChecking=no -i '$SSH_KEY' \
                                "target/*.war" \
                                ansible@10.10.10.229:"${REMOTE_ARTIFACT_DIR}/"
                        """
                        
                        // Transfer monitoring playbooks
                        sh """
                            scp -o StrictHostKeyChecking=no -i '$SSH_KEY' \
                                "ansible/playbooks/prometheus_deploy.yml" \
                                ansible@10.10.10.229:"${ANSIBLE_HOME}/playbooks/"
                        """
                    }
                }
            }
        }
        
        // STAGE 5: Build Docker Image
        stage('Build Application') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 "
                            cd ${ANSIBLE_HOME} && \
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/docker_build.yml \
                                --extra-vars 'artifact_path=${REMOTE_ARTIFACT_DIR}/*.war image_tag=${BUILD_NUMBER}'
                        "
                    """
                }
            }
        }
        
        // STAGE 6: Deploy Application to K8s
        stage('Deploy Application') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 "
                            cd ${ANSIBLE_HOME} && \
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/k8s_deploy.yml \
                                --extra-vars 'image_tag=${BUILD_NUMBER}'
                        "
                    """
                }
            }
        }
        
        // STAGE 7: Deploy Monitoring Stack
        stage('Deploy Monitoring') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 "
                            cd ${ANSIBLE_HOME} && \
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/prometheus_deploy.yml \
                                --extra-vars '\
                                    prometheus_version=${PROMETHEUS_VERSION} \
                                    grafana_version=${GRAFANA_VERSION} \
                                    app_name=myapp \
                                    app_port=8080'
                        "
                    """
                }
            }
        }
        
        // STAGE 8: Verify Deployment
        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        // Verify application
                        def appStatus = sh(script: """
                            ssh -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get pods,svc -l app=myapp -n default
                            '
                        """, returnStdout: true)
                        
                        // Verify monitoring
                        def monitoringStatus = sh(script: """
                            ssh -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get pods,svc -l app=monitoring -n monitoring
                            '
                        """, returnStdout: true)
                        
                        echo "=== Application Status ===\n${appStatus}"
                        echo "=== Monitoring Status ===\n${monitoringStatus}"
                        
                        // Get Grafana NodePort
                        def grafanaUrl = sh(script: """
                            ssh -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get svc grafana -n monitoring -o jsonpath="{.spec.ports[0].nodePort}"
                            '
                        """, returnStdout: true).trim()
                        
                        echo "Grafana Dashboard: http://<your-node-ip>:${grafanaUrl} (admin/admin)"
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Cleanup temporary SSH key
            withCredentials([sshUserPrivateKey(
                credentialsId: 'Ans2-ssh-key',
                keyFileVariable: 'SSH_KEY'
            )]) {
                sh """
                    ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                        rm -f ~/.ssh/jenkins_key
                    ' || true
                """
            }
            cleanWs()
        }
        success {
            slackSend color: 'good', message: "SUCCESS: Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            slackSend color: 'danger', message: "FAILED: Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
