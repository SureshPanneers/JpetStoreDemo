pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/venkattharakram/JPetStore.git'
        BRANCH = 'main'
        DOCKER_IMAGE = 'tharak397/ciscodevops'
        SONAR_PROJECT_KEY = 'JPetStore'
        SONAR_HOST_URL = 'http://192.168.49.1:9000/'
        SONAR_TOKEN = credentials('17f04bf5-a0b6-4e5c-a999-7ee0f8a9ddbe') // Jenkins credential ID
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds' // Jenkins credential ID
        DEPLOYMENT_FILE = 'deployment.yaml'
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
            steps {
                withSonarQubeEnv('My_SonarQube_Server') {
                    sh 'mvn sonar:sonar'
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
              withCredentials([file(credentialsId: 'minikube-kube-config', variable: 'KUBECONFIG')]) {
            sh 'kubectl apply -f ${DEPLOYMENT_FILE}'
        }
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
