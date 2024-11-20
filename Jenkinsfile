pipeline {
    agent any

    tools {
        jdk 'jdk'
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_CREDENTIALS = credentials('docker-cred') // Docker credentials
    }

    stages {
        stage('Git-Checkout') {
            steps {
                script {
                    echo "Branch being built: ${env.BRANCH_NAME}"
                    checkout scm
                }
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonarqube') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.url=http://52.87.207.241:9000/ \
                            -Dsonar.login=squ_776d93de9a16828382810b5445de938b6a44838a \
                            -Dsonar.projectName=ekart-${env.BRANCH_NAME} \
                            -Dsonar.java.binaries=. \
                            -Dsonar.projectKey=ekart-${env.BRANCH_NAME}'''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
        }

        stage('Docker Build and Push') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker-cred', url: '']) {
                        def dockerImage = "pravinhcl/ekart:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                        sh "docker build -t ${dockerImage} -f docker/Dockerfile ."
                        sh "docker push ${dockerImage}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Dynamically determine the Docker image to pull
                    def dockerImage = "pravinhcl/ekart:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

                    withDockerRegistry([credentialsId: 'docker-cred', url: '']) {
                        // Pull the Docker image
                        sh "docker pull ${dockerImage}"

                        // Stop and remove any running container named 'ekart'
                        sh 'docker stop ekart || true'
                        sh 'docker rm ekart || true'

                        // Run the Docker container using the pulled image
                        sh "docker run -d --name ekart -p 8070:8070 ${dockerImage}"
                    }
                }
            }
        }
    }
}
