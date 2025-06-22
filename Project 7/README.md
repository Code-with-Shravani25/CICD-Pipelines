# CI/CD Pipeline: Jenkins to Kubernetes Deployment

This project demonstrates a complete CI/CD pipeline where a Java-based web application is built with Maven, containerized with Docker, pushed to Docker Hub, and deployed to an AWS EKS (Kubernetes) cluster using Jenkins.

---

## 🛠️ Prerequisites

- Two Ubuntu-based EC2 instances (Jenkins EC2 and Kubernetes EC2)
- IAM role with the following permissions:
  - `IAMFullAccess`
  - `EC2FullAccess`
  - `VPCFullAccess`
  - `CloudFormationFullAccess`
  - `AdministratorAccess`
- Docker Hub account (for pushing images)
- GitHub repository with application source code
- Internet access and basic Linux knowledge
- Port 8080 open on Jenkins EC2 security group

---

## 🖥️ Step 1: Setup Jenkins EC2 Instance

### Install Java, Jenkins, and Maven

```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
```

#### Jenkins Installation

```bash
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

#### Maven Installation

```bash
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.tar.gz
tar -xzvf apache-maven-3.9.10-bin.tar.gz
ln -s /opt/apache-maven-3.9.10 /opt/maven
```

##### Configure Maven Environment

```bash
sudo vi /etc/profile.d/maven.sh
```

Add:

```bash
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}
```

Apply:

```bash
chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
```

### Install Docker

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

---

## ☸️ Step 2: Setup Kubernetes EC2 Instance

### Install Docker

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

### Install AWS CLI

```bash
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Install kubectl

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

### Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## 🔐 Step 3: IAM Role Setup for Kubernetes EC2

1. Go to AWS IAM → Create Role → Use Case: EC2  
2. Attach permissions:
   - `IAMFullAccess`
   - `EC2FullAccess`
   - `VPCFullAccess`
   - `CloudFormationFullAccess`
   - `AdministratorAccess`
3. Attach the role to your Kubernetes EC2 instance

---

## ☁️ Step 4: Create EKS Cluster

```bash
eksctl create cluster \
--name my-cluster \
--region ap-south-1 \
--nodegroup-name linux-nodes \
--node-type t3.medium \
--nodes 2 \
--nodes-min 1 \
--nodes-max 3 \
--managed
```

---

## 🔧 Step 5: Jenkins Docker Permission

Run on Kubernetes EC2:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## 🧩 Step 6: Jenkins Configuration

### Install Plugins

- Docker
- Kubernetes CLI
- SSH Pipeline Steps
- Maven Integration
- SSH Agent

### Global Tool Configuration

- **Maven**  
  - Name: `Maven3`  
  - Path: `/opt/maven`

- **Java**  
  - Name: `java21`  
  - Path: `/usr/lib/jvm/java-21-openjdk-amd64`

---

## 🔐 Step 7: SSH Setup Between Jenkins EC2 and K8s EC2

### On Jenkins EC2:

```bash
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub
```

### On Kubernetes EC2:

```bash
vi ~/.ssh/authorized_keys
# Paste the Jenkins public key
```

### Test Connection:

```bash
ssh ubuntu@<k8s-ec2-private-ip>
```

---

## 🔐 Step 8: Credentials in Jenkins

Go to: **Manage Jenkins → Credentials**

1. **DockerHub Credentials**  
   - ID: `dockerhub-creds`  
   - Username and Password

2. **SSH Credentials**  
   - Username: `ubuntu`  
   - Private Key: `~/.ssh/id_rsa`  
   - ID: `ssh-key-id`

---

## 🔁 Step 9: Jenkins Pipeline Script

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'shravani2001/webapp'
        DOCKER_TAG = 'latest'
        K8S_SERVER = 'ubuntu@<k8s-ec2-private-ip>'
        APP_NAME = 'webapp'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/Code-with-Shravani25/Project3.git'
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn clean install package'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$DOCKER_TAG .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push $DOCKER_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent (credentials: ['ssh-key-id']) {
                    sh '''
                        scp deployment.yml service.yml $K8S_SERVER:/home/ubuntu/
                        ssh $K8S_SERVER "kubectl apply -f deployment.yml"
                        ssh $K8S_SERVER "kubectl apply -f service.yml"
                    '''
                }
            }
        }
    }
}
```
---
---

## 📚 References

- [Jenkins Official Site](https://www.jenkins.io/)
- [Apache Maven](https://maven.apache.org/)
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [AWS EKS with eksctl](https://eksctl.io/)
- [AWS CLI Installation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html)

## ✅ Outcome

By the end of this project, you will have:

- A fully functional Jenkins server with Maven, Java, and Docker configured.
- A Kubernetes (EKS) cluster up and running on AWS.
- An automated Jenkins pipeline that:
  - Pulls code from GitHub
  - Builds a WAR package using Maven
  - Builds a Docker image and pushes it to Docker Hub
  - Deploys the image to the EKS cluster using `kubectl`

---

