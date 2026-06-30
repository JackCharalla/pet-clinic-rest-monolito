pipeline {
    agent {
        docker {
            image 'maven:3.9.15-amazoncorretto-21'
        }
    }
    environment {
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
        SONAR_USER_HOME = "${WORKSPACE}/.sonar"
        ARTIFACTORY_CREDENTIALS = credentials('artifactory-credentials')
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile -B -ntp'
            }
        }
        stage('Tests (Junit + Jacoco)') {
            steps {
                sh 'mvn test jacoco:report -B -ntp'
            }
            post {
                success {
                    junit 'target/surefire-reports/*.xml'
                    recordCoverage(tools: [[parser: 'JACOCO']])
                }
            }
        }
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests -B -ntp'
            }
        }
        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonarqube'){
                    script {
                        if (env.CHANGE_ID) {
                            sh """
                                mvn sonar:sonar -B -ntp \
                                -Dsonar.pullrequest.key=${env.CHANGE_ID} \
                                -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} \
                                -Dsonar.pullrequest.base=${env.CHANGE_TARGET}
                            """
                        } else {
                            def branchName = GIT_BRANCH.replaceFirst('^origin/', '')
                            println "Branch name: ${branchName}"
                            sh "mvn sonar:sonar -B -ntp -Dsonar.branch.name=${branchName} -Dsonar.branch.target=${branchName}"
                        }
                    }
                }
            }
        }
        stage('Artifactory') {
            steps {
                  withCredentials([file(credentialsId: 'artifactory-settings', variable: 'M2_SETTINGS')]) {
                    sh 'mvn clean package -B -DskipTests -s ${M2_SETTINGS}'
                }                

                script{
                    def releaseRepo = 'spring-petclinic-rest-release'
                    def snapshotRepo = 'spring-petclinic-rest-snapshot'
                    
                    def server = Artifactory.server 'artifactory'
                    
                    def pom = readMavenPom file: 'pom.xml'
                    println pom.groupId

                    def groupIdPath = pom.groupId.replaceAll("\\.", "/")
                    println groupIdPath

                    def uploadSpec = """
                        {
                            "files": [
                                {
                                    "pattern": "target/.*.jar",
                                    "target": "${releaseRepo}/${groupIdPath}/${pom.artifactId}/${pom.version}/",
                                    "regexp": "true",
                                    "props": "build.url=${RUN_DISPLAY_URL};build.user=${USER}"
                                }
                            ]
                        }
                    """
                    server.upload spec: uploadSpec
                }
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