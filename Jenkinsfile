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
        stage('Docker Deployment') {
            steps {
                dir('ansible') {  // Relative to java_build/
                    ansiblePlaybook(
                        playbook: 'playbooks/docker.yml',
                        inventory: 'inventories/production/hosts'
                    )
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
