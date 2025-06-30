pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        ANSIBLE_HOME = '/home/ansible/ansible'
        BUILD_NUMBER = "${env.BUILD_ID}"
        PROMETHEUS_VERSION = 'v2.47.2'
        GRAFANA_VERSION = '10.2.3'
        ARTIFACT_NAME = "ABCtechnologies-${BUILD_NUMBER}.war"
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
                archiveArtifacts artifacts: "target/*.war", fingerprint: true
            }
        }

        // STAGE 3: Transfer Artifacts
        stage('Transfer WAR File') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        scp -o StrictHostKeyChecking=no -i '$SSH_KEY' \
                            "target/*.war" \
                            ansible@10.10.10.229:"${ANSIBLE_HOME}/tmp/"
                    """
                }
            }
        }

        // STAGE 4: Docker Build
        stage('Build Docker Image') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    ),
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                            cd ${ANSIBLE_HOME} && \
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/docker_build.yml \
                                --extra-vars "\
                                    artifact_path=tmp/*.war \
                                    docker_registry=your-registry \
                                    docker_user=${DOCKER_USER} \
                                    docker_pass=${DOCKER_PASS} \
                                    image_tag=${BUILD_NUMBER}"
                        '
                    """
                }
            }
        }

        // STAGE 5: Kubernetes Deployment
        stage('Deploy to Kubernetes') {
            steps {
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
                                --extra-vars "\
                                    image_name=your-registry/johnberb/myapp \
                                    image_tag=${BUILD_NUMBER}"
                        '
                    """
                }
            }
        }

        // STAGE 6: Deploy Monitoring Stack
        stage('Deploy Monitoring') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'Ans2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i '$SSH_KEY' ansible@10.10.10.229 '
                            # Create namespace if not exists
                            kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
                            
                            # Deploy Prometheus and Grafana
                            cd ${ANSIBLE_HOME} && \
                            ansible-playbook \
                                -i /etc/ansible/hosts \
                                playbooks/prometheus_deploy.yml \
                                --extra-vars "\
                                    prometheus_version=${PROMETHEUS_VERSION} \
                                    grafana_version=${GRAFANA_VERSION} \
                                    app_service=myapp.default.svc.cluster.local:8080"
                        '
                    """
                }
            }
        }

        // STAGE 7: Verify Deployment
        stage('Verify') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'Ans2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        def appStatus = sh(script: """
                            ssh -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get pods,svc -l app=myapp -n default
                            '
                        """, returnStdout: true)
                        
                        def monitoringStatus = sh(script: """
                            ssh -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get pods,svc -n monitoring
                            '
                        """, returnStdout: true)
                        
                        echo "=== Application Status ===\n${appStatus}"
                        echo "=== Monitoring Status ===\n${monitoringStatus}"
                        
                        def grafanaUrl = sh(script: """
                            ssh -i '$SSH_KEY' ansible@10.10.10.229 '
                                kubectl get svc -n monitoring -l app=grafana -o jsonpath="{.items[0].spec.ports[0].nodePort}"
                            '
                        """, returnStdout: true).trim()
                        
                        echo "Grafana Dashboard: http://<your-node-ip>:${grafanaUrl}"
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
            slackSend color: 'good', 
                      message: "SUCCESS: Build ${BUILD_NUMBER} deployed with monitoring"
        }
        failure {
            slackSend color: 'danger', 
                      message: "FAILED: Build ${BUILD_NUMBER}"
        }
    }
}