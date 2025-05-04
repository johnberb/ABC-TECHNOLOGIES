pipeline {
    agent any
    
    tools {
                maven 'maven3'  
    }
    
    //environment {
        
        //JAVA_HOME = "${tool 'JDK8'}"  // If you need specific JDK
    //}
    
    stages {
        stage('Checkout') {
            steps {
                
                git branch: 'master',  
                     url: 'https://github.com/johnberb/ABC-TECHNOLOGIES.git'
                      
            }
        }
        
        stage('Build & Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'mvn test'
                
                
                junit '**/target/surefire-reports/*.xml'
                
                
                jacoco execPattern: '**/target/jacoco.exec',
                      classPattern: '**/target/classes',
                      sourcePattern: '**/src/main/java'
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package'
                
                
                archiveArtifacts artifacts: '**/target/*.war', 
                                fingerprint: true
            }
        }
        stage('Run Ansible') {
            steps {
                sshagent(['ansible-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ansible@10.10.10.229 \
                    'cd /path/to/ansible/project && \
                    ansible-playbook -i inventories/production playbooks/deploy.yml \
                    --extra-vars "jenkins_build=${BUILD_NUMBER}"'
                    """
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
