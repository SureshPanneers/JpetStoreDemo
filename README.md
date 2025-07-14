# Cisco Demo Project Documentation for WAR Deployment to Minikube Clusters

---

## 🛠️ Prerequisites and Configuration

### Install Required Packages on Jenkins Server

#### ✅ Java Installation
```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

#### ✅ Jenkins Installation
```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

---

### 🐳 Docker Installation
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

docker --version
docker compose version
```

---

### 🔍 SonarQube Installation
```bash
sudo docker pull sonarqube
sudo docker run -d --name sonarqube -p 9000:9000 sonarqube
```

---

### ⚙️ Kubectl CLI Installation
```bash
sudo apt update
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

---

### 🌱 Minikube Installation
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl apt-transport-https ca-certificates conntrack
wget -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
minikube start
```

---

## 🔀 Jenkins CI/CD Process

### 🔌 Required Plugins
- Docker Pipeline
- Pipeline
- Pipeline View
- SonarQube Scanner
- Kubectl CLI

### 🌐 Global Tool Configuration
**SonarQube:**
- **Name:** `SonarQube`
- **Server URL:** `http://<SonarQube-IP>:9000`
- **Token:** Stored as Jenkins Credentials (Secret Text)

---

## 📄 CI/CD Pipeline Files Explanation

### 1. 🐳 Dockerfile
```Dockerfile
FROM tomcat:9.0-jdk11
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/maven-wrapper.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
```
**Explanation:**
- Uses Tomcat with JDK 11 as base image.
- Removes default webapps.
- Deploys the WAR as `ROOT.war` so it runs at `/` context path.

---

### 2. 📆 `deployment.yaml`
```yaml
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
```
**Explanation:**
- Deploys a container from Docker Hub image to Kubernetes.
- Namespace: `jpetstore`, 1 replica, exposed on port 8080.

---

### 3. 🌐 `service.yaml`
```yaml
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
```
**Explanation:**
- Exposes the app on port 30007 so it's accessible externally.
        

---

### 4. 🧪 `Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        GIT_REPO = 'https://github.com/venkattharakram/JPetStore.git'
        BRANCH = 'main'
        DOCKER_IMAGE = 'tharak397/ciscodevops'
        SONAR_PROJECT_KEY = 'JPetStore'
        SONAR_HOST_URL = 'http://192.168.0.11:9000/'
        SONAR_TOKEN = credentials('17f04bf5-a0b6-4e5c-a999-7ee0f8a9ddbe')
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
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
                    sh "sed -i 's|image: .*|image: ${imageTag}|' ${DEPLOYMENT_FILE}"
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
            echo "✅ Deployment successful to Minikube"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
```

**Explanation:**

- **Checkout Code**: Pulls code from GitHub.
- **Compile Code**: Builds the WAR file.
- **Build Docker Image**: Builds image with unique tag.
- **Push Docker Image**: Pushes image to Docker Hub.
- **Update Deployment File**: Replaces image tag in YAML.
- **Deploy to Minikube**: Deploys the updated app.

### ✅ Jenkins Pipeline Overview:

<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-22-05" src="https://github.com/user-attachments/assets/49674db6-7c35-42b4-8053-297e589f269d" />

### ✅ After Pipeline Execution :
<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-24-26" src="https://github.com/user-attachments/assets/42062cc5-ccaf-410c-8210-be8704910276" />

###  🐳 Docker Hub Image:

<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-26-47" src="https://github.com/user-attachments/assets/70c421ff-a9c0-4690-99c1-50893e5022b6" />

## 📦 Pods and Service Status:

<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-28-23" src="https://github.com/user-attachments/assets/dde688c8-7dde-4fb0-9067-7901565c81c6" />

### 🌐 Application Home Page:
<img width="1920" height="1080" alt="Screenshot from 2025-07-14 22-29-07" src="https://github.com/user-attachments/assets/7fd30bc9-7232-47ec-a3e7-64fed490a179" />






