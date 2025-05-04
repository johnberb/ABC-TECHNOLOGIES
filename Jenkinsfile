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
                sshagent(['ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCWx/BuVVPPU26AV60hjqEq6KSQRDV0zYq5ATRhPAMbebMs+uhR3RHBh/6JXPAj3YtQEdYfbiVm8ZpG/p6GUzkaLBhb8LkWCpiUh64zSNWMKKROROpFbFmfmr+j82/Vqw88G6RT8i2Vhf7aXnMb3rjuFOmcd2B1ucZS5JIbVByiVGP0BbcYOuqxQ0vNUCcn0b/nNp1XqzbMHGI/wdEn9s3spm70dnhSkEJZzQ+cAE900gQEJHqJsdoP9UjFrSCO8djlj3ftM3X2cQAytNE4uBJdVLxAyNRiTLF73zPqCLM/envoKVvaJBN6ayK5MOatly5/0pfZid2BWU7i94RRl/IBLmoi3KcZFWAEGhybMkE7dhDx9TcTvOIDwrTi66G6oR0gpl2YwXcjFXNJ7xKdCSUHtOeAP66Qw3+PsjqB1UKJlFgv78YTPVrXc+KN71n+E2/swQI6UHfLzTx0l4j8eZKY3DCR71RF198pxrYj17k34bAef5R75fTnIiC4vkTI2IXO1b044ER5RvU/AhgnzZfKDLQiOZUN5dGkSOIF+j0YVyHI7KL0aurZKXCdFNf3pWZVvUFwmIkYtwAQ/YfAE9jtK6hhCSaZS8E0iDSoJ6tw99VqsPrkuRXXvjVXGC23e82uPKysHistfclNtKLxjQvhYn5HFQJDc6qh4ScYLbpo4w== jenkins@ip-10-10-10-223.ap-south-1.compute.internal ']) {
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
