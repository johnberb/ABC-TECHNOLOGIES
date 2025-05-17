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
        stage('Deploy Dockerfile via SSH') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'Ansible', 
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'Dockerfile', // File to send
                                    //removePrefix: 'Dockerfile', // Optional: adjust path if needed
                                    remoteDirectory: '/opt',   // Remote target directory
                                    execCommand: ''            // Optional: post-transfer command
                                )
                            ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: true
                        )
                    ]
                )
            }
        }
        
        
    }
    
    post {
        always {
            
            cleanWs()
        }
        
    }
}
