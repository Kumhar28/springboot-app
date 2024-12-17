pipeline {
    agent any

    tools {
        maven 'maven3' // Ensure 'maven3' is configured in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        registry = "869935094555.dkr.ecr.us-east-1.amazonaws.com/myd"
        imageName = "myd"
        SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:my-topic'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/Kumhar28/springboot-app.git'
            }
        }
        stage('Build Jar') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage ("Build") {
            steps {  
                sh 'mvn package'
        }
    }        
        stage ("Build Image") {
            steps {  
                sh 'docker build -t ${imageName}:latest .'
                sh 'docker images'
        }
    }
        stage ("Trivy scan") {
            steps {  
                sh "trivy image --format table -o trivy-image-report.html ${imageName}:latest"
        }
    }    
        stage ("Push to ECR") {
            steps {
                script {
                    sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 869935094555.dkr.ecr.us-east-1.amazonaws.com"
                    sh 'docker tag ${imageName}:latest $registry:latest'
                    sh 'docker push $registry:latest'
                }
            }
        } 
        stage('Deploy to Dev') {
            steps { 
                script {
                 sh "kubectl apply -f deployment.yaml" 
                } 
            } 
        }         
        stage('Deploy to QA') {
            steps { 
                script {
                 sh "kubectl apply -f deployment.yaml" 
                } 
            } 
        } 
        stage('Approval for Production') { 
            steps { 
                script { 
                    input message: 'Approve deployment to production?', ok: 'Deploy' 
                } 
            }
        } 
        stage('Deploy to Production') { 
            steps { 
                script { 
                    sh "kubectl apply -f deployment.yaml"
                } 
            } 
        }
        post {
             always { 
                junit 'target/surefire-reports/*.xml' 
                sonarQubeQualityGate() 
                
                script { 
                    if (currentBuild.result == 'FAILURE' || currentBuild.result == 'UNSTABLE') { 
                        sh "aws sns publish --topic-arn $SNS_TOPIC_ARN --message 'Build failed or unstable'"
                         } 
                } 
                script { 
                    def coverage = sh(script: 'mvn sonar:sonar | grep "Coverage"', returnStdout: true).trim() 
                    if (coverage < 80) { 
                        sh "aws sns publish --topic-arn $SNS_TOPIC_ARN --message 'Coverage less than 80%'" 
                        error('Coverage less than 80%') 
                        } 
                } 
            }
        }    
    }           
}
