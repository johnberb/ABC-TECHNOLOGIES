pipeline {
    agent any
    
    tools {
                maven 'maven'  
    }
    
    //environment {
        
        //JAVA_HOME = "${tool 'JDK8'}"  // If you need specific JDK
    //}
    
    stages {
        stage('Checkout') {
            steps {
                
                git branch: 'master',  
                     url: 'https://github.com/johnberb/ABC-TECHNOLOGIES.git',
                      
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
        
        
    }
    
    post {
        always {
            
            cleanWs()
        }
        success {
            
            mail to: 'rubengs.j@gmail.com',
                 subject: "Build Success: ${currentBuild.fullDisplayName}",
                 body: "Build ${currentBuild.result}: ${env.BUILD_URL}"
        }
        failure {
            
            mail to: 'rubengs.j@gmail.com',
                 subject: "Build Failed: ${currentBuild.fullDisplayName}",
                 body: "Build ${currentBuild.result}: ${env.BUILD_URL}"
        }
    }
}