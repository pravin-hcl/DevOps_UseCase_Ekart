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
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
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
                        // Fetch the short commit ID
                        def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

                        // Define the image tag using branch name and commit ID
                        def tag = "latest-${env.BRANCH_NAME}-${commitId}"
                        def imageName = "pravinhcl/ekart:${tag}"

                        // Build the Docker image with the tag
                        sh "docker build -t ${imageName} -f docker/Dockerfile ."

                        // Push the image with the tag
                        sh "docker push ${imageName}"
                    }
                }
            }
        }

        stage('Pull and Deploy') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker-cred', url: '']) {
                        // Fetch the short commit ID
                        def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

                        // Define the image tag using branch name and commit ID
                        def tag = "latest-${env.BRANCH_NAME}-${commitId}"
                        def imageName = "pravinhcl/ekart:${tag}"

                        // Pull the Docker image
                        sh "docker pull ${imageName}"

                        // Stop and remove any running container with the same name
                        sh 'docker stop ekart || true'
                        sh 'docker rm ekart || true'

                        // Run the Docker container with the pulled image
                        sh "docker run -d --name ekart -p 8070:8070 ${imageName}"
                    }
                }
            }
        }
    }
}
