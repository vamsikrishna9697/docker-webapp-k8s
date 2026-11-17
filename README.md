pipeline-
pipeline {
    agent any
    tools {
        maven 'mymaven'
    }

    stages {
        stage('code') {
            steps {
                git branch: 'main', url: 'https://github.com/vamsikrishna9697/docker-webapp-k8s.git'
            }
        }
        stage('Maven') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('CQA') {
            steps {
                

withSonarQubeEnv('mysonar') {
sh "mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=myproject"
              }
            }
        }
        stage('QualityGates') {
            steps {
                waitForQualityGate abortPipeline: true, credentialsId: 'sonar-cred'
            }
        }
        stage('Artifact') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'vprofile', classifier: '', file: 'target/vprofile-v2.war', type: 'war']], credentialsId: 'nexus', groupId: 'com.visualpathit', nexusUrl: '18.224.71.90:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'myrepo', version: 'v2'
            }
        }
        stage('Images') {
            steps {
                sh 'docker build -t appimage .'
                sh 'docker build -t dbimage Docker-db'
            }
        }
        stage('Scan') {
            steps {
                sh 'trivy image appimage >> app-image-reports.txt'
                sh 'trivy image dbimage >> db-image-reports.txt'
            }
        }
        stage('Registry') {
            steps {
                script {
withDockerRegistry(credentialsId: 'docker hub') {
    sh 'docker tag appimage vamc76/k8swebapp:frontend'
    sh 'docker tag dbimage vamc76/k8swebapp:db'
    sh 'docker push vamc76/k8swebapp:frontend'
    sh 'docker push vamc76/k8swebapp:db'
                   }
                }
            }
        }
        stage('Compose') {
            steps {
                sh 'docker-compose build'
                sh 'docker-compose up -d'
                sleep 10
                sh 'docker ps'
            }
        }
    }
}
