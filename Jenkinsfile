pipeline {
    agent any
    tools {
        maven 'maven3.9.14'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile -B -ntp'
            }
        }
        stage('Test Junit') {
            steps {
                sh 'mvn test -B -ntp'
            }
            post { 
                success {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Test Jacoco') {
            steps {
                sh 'mvn jacoco:report -B -ntp'
            }
            post { 
                success {
                    recordCoverage(tools: [[parser: 'JACOCO']])
                }
            }
        }
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests -B -ntp'
            }
        }
    }
    post { 
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
        cleanup {
            cleanWs()
        }
    }
}