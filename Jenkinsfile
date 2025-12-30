pipeline {
    agent any
    tools {
        maven "Maven 3.9.9"
        jdk "JDK 17"
    }


    environment {
        registryCredential = 'ecr:us-east-1:awscreds'
        appRegistry = "860552602566.dkr.ecr.us-east-1.amazonaws.com/vprofile-app-image"
        vprofileRegistry = "https://860552602566.dkr.ecr.us-east-1.amazonaws.com"
        cluster = "vprofile"
        service = "vprofile-app-svc"
    }
  stages {
   
        stage('Fetch code') {
            steps {
               git branch: 'docker', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }

        }


        stage('Build'){
            steps{
               sh 'mvn install -DskipTests'
            }

            post {
               success {
                  echo 'Now Archiving it...'
                  archiveArtifacts artifacts: '**/target/*.war'
               }
            }
        }

        stage('UNIT TEST') {
            steps{
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps{
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage("Sonar code analysis") {
            environment {
                scannerHome = tool 'Sonarqube6.2'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                sh """${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=target/classes \
                    -Dsonar.junit.reportsPath=target/surefire-reports \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                """
                }
            }
        }

        stage('Build App Image') {
          steps {
       
            script {
                dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
          }
    
        }

        stage('Upload App Image') {
          steps{
            script {
              docker.withRegistry( vprofileRegistry, registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }

        stage('Remove Container Images from Jenkins'){
            steps{
                sh 'docker rmi -f $(docker images -a -q)'
            }
        }


        stage('Deploy to ecs') {
          steps {
            withAWS(credentials: 'awscreds', region: 'us-east-1') {
            sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
               }
          }
        }

  }
}
