
# 🚀 CI/CD Project: GitHub → Jenkins → Docker → Docker Hub (Push and Pull)

This project sets up an automated CI/CD pipeline that pulls source code from GitHub, builds and packages it with Jenkins, and deploys it inside a Docker container. The built Docker image is pushed to Docker Hub and run on a Docker-enabled EC2 instance.

---

## 📦 Project Flow

```
GitHub (Source Code)
   ⮕ Jenkins (Build + Test + Push to Docker Host)
      ⮕ Docker Host (Build Image + Push to Docker Hub + Deploy Container)
```

---

## 🧱 Jenkins & Maven Setup (on EC2)

Install Java, Jenkins, and Maven:

```bash
sudo apt update -y
sudo apt install openjdk-21-jdk -y

# Jenkins
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

# Maven
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.tar.gz
tar -xzvf apache-maven-3.9.10-bin.tar.gz
ln -s /opt/apache-maven-3.9.10 /opt/maven
```

Configure environment variables:

```bash
vi /etc/profile.d/maven.sh
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}

chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
```

---

## ⚙️ Jenkins Configuration

- Access Jenkins: `http://<Jenkins-EC2-IP>:8080`
- Initial password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install plugins:
- Maven Integration Plugin
- Publish Over SSH Plugin
- Deploy to Container Plugin

Global Tool Configuration:
- JDK:
  - Name: `java21`
  - Path: `/usr/lib/jvm/java-21-openjdk-amd64`
- Maven:
  - Name: `Maven3`
  - Path: `/opt/maven`

---

## 🐳 Docker Host EC2 Setup

Install Docker and create user:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker

sudo su
useradd ansadmin
passwd ansadmin
usermod -aG docker ansadmin
visudo
# Add:
ansadmin ALL=(ALL) NOPASSWD: ALL
```

Setup Docker build directory and permissions:

```bash
sudo su - ansadmin
mkdir -p /opt/docker
sudo chown -R ansadmin:ansadmin /opt/docker
sudo chmod -R 755 /opt/docker
sudo mkdir -p /home/ansadmin
sudo chown ansadmin:ansadmin /home/ansadmin
```

---

## 🛠️ Jenkins Pipeline Job

Create a **Pipeline** job and use the following declarative pipeline:

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'java21'
    }

    stages {
        stage('Clone Source Code') {
            steps {
                git 'https://github.com/Code-with-Shravani25/Project2.git'
            }
        }

        stage('Build and Package') {
            steps {
                sh 'mvn clean install package'
            }
        }

        stage('Publish over SSH and Deploy') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'Docker',
                        transfers: [
                            sshTransfer(
                                sourceFiles: 'webapp/target/*.war',
                                removePrefix: 'webapp/target',
                                remoteDirectory: '/opt/docker'
                            ),
                            sshTransfer(
                                sourceFiles: 'Dockerfile',
                                remoteDirectory: '/opt/docker'
                            ),
                            sshTransfer(
                                execCommand: '''
cd /opt/docker
docker stop demo || true
docker rm demo || true
docker build -t demo .
docker tag demo shravani2001/demo
docker push shravani2001/demo
docker rmi demo shravani2001/demo
docker run -d --name demo -p 8090:8080 shravani2001/demo
''',
                                execTimeout: 120000
                            )
                        ]
                    )
                ])
            }
        }
    }
}
```

---

## 🔍 Validation & Testing

1. Trigger Jenkins job.
2. Check Docker images on Docker Hub.
3. Verify Docker container is running on Docker Host:
```bash
docker ps -a
```
4. Access application:
```
http://<Docker_Public_IP>:8090/webapp-1.0-SNAPSHOT/
```

---

## ✅ Outcome

- Code is pulled from GitHub.
- Jenkins builds and packages WAR.
- Docker image is built and pushed to Docker Hub.
- Image is deployed and run on Docker Host.

---

## 📎 References

- [Jenkins Official Docs](https://www.jenkins.io/doc/)
- [Docker Hub](https://hub.docker.com/)
- [Apache Maven](https://maven.apache.org/)
- [Tomcat Docker Image](https://hub.docker.com/_/tomcat)

---

## 📌 Notes

- Ensure Dockerfile is in the GitHub repo.
- Docker Host must have Docker running and open port `8090`.
- Docker Hub account (`shravani2001`) credentials must be valid.
- Jenkins must have SSH access to Docker Host (`ansadmin`).
