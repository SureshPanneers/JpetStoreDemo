# Cisco Demo project Documentation for WAR Deployment to Minikube clusters


## Prerequisites and Configuration

#### Install below Packages on Jenkins server

**Install Java:**
sudo apt update
sudo apt install openjdk-17-jdk -y

**Install Jenkins:**

wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y

# start Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins


### Docker installation >>>>>>>>>>>>>>

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg


sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

docker --version
docker compose version


### Sonarqube  installation >>>>>>>>>>>>>>

sudo docker pull sonarqube

sudo docker run -d --name sonarqube   -p 9000:9000   sonarqube



### Kubectl CLI  installation >>>>>>>>>>>>>>


sudo apt update

curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

kubectl version --client


### Miniqube installation>>>>>>>

sudo apt update && sudo apt upgrade -y
sudo apt install -y curl apt-transport-https ca-certificates conntrack
wget -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
minikube start


### Install plugins
- Docker Pipeline
- Pipeline
- Pipeline view
- SonarQube Scanner
- kubectl cli

### Global tool configuration

**SonarQube:**
- Name: SonarQube
- Server URL: http://<SonarQube-IP>:9000
- Token: Add via Jenkins Credentials (Secret Text)


## Docker file >>>>>

FROM tomcat:9.0-jdk11

RUN rm -rf /usr/local/tomcat/webapps/*

COPY target/maven-wrapper.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080


### Deployment.yaml file>>>>>

apiVersion: apps/v1
kind: Deployment
metadata:
  name: jpetstore-deployment
  namespace: jpetstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jpetstore
  template:
    metadata:
      labels:
        app: jpetstore
    spec:
      containers:
      - name: jpetstore
        image: tharak397/ciscodevops:2.0
        ports:
        - containerPort: 8080
        
        
##service.yaml file >>>>

apiVersion: v1
kind: Service
metadata:
  name: jpetstore-service
spec:
  type: NodePort
  selector:
    app: jpetstore
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080     
      nodePort: 30007
        

## 4. Jenkinsfile

pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/venkattharakram/JPetStore.git'
        BRANCH = 'main'
        DOCKER_IMAGE = 'tharak397/ciscodevops'
        SONAR_PROJECT_KEY = 'JPetStore'
        SONAR_HOST_URL = 'http://192.168.0.11:9000/'
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

   /*     stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarQube Scanner' // name configured in Jenkins tools
            }
            steps {
                withSonarQubeEnv('My SonarQube Server') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.sources=src -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_TOKEN}"
                }
            }
        } */

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

##### Jenkins pipeline overview 

<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-22-05" src="https://github.com/user-attachments/assets/49674db6-7c35-42b4-8053-297e589f269d" />

### After pipeline execution 
<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-24-26" src="https://github.com/user-attachments/assets/42062cc5-ccaf-410c-8210-be8704910276" />

###  Docker hub images

<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-26-47" src="https://github.com/user-attachments/assets/70c421ff-a9c0-4690-99c1-50893e5022b6" />

## Pods and service running in server 

<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-28-23" src="https://github.com/user-attachments/assets/dde688c8-7dde-4fb0-9067-7901565c81c6" />

### application home page url 
<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-29-07" src="https://github.com/user-attachments/assets/7fd30bc9-7232-47ec-a3e7-64fed490a179" />






