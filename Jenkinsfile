pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/your-org/your-repo.git'
        BRANCH = 'main'
        DOCKER_IMAGE = 'your-dockerhub-username/your-image-name'
        SONAR_PROJECT_KEY = 'your-project-key'
        SONAR_HOST_URL = 'http://your-sonarqube-url'
        SONAR_TOKEN = credentials('sonar-token') // Jenkins credential ID
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds' // Jenkins credential ID
        DEPLOYMENT_FILE = 'k8s/deployment.yaml'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: "${BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Compile Code') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarQube Scanner' // name configured in Jenkins tools
            }
            steps {
                withSonarQubeEnv('My SonarQube Server') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.sources=src -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_TOKEN}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                        sh "docker push ${imageTag}"
                    }
                }
            }
        }

        stage('Update Deployment File') {
            steps {
                script {
                    def imageTag = "${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                    sh """
                        sed -i 's|image: .*|image: ${imageTag}|' ${DEPLOYMENT_FILE}
                    """
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh 'kubectl apply -f ${DEPLOYMENT_FILE}'
            }
        }
    }

    post {
        success {
            echo "Deployment successful to Minikube"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
