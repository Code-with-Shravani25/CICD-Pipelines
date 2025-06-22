
# 🚀 CI/CD Project 3: Git → Jenkins → Docker Deployment

This project demonstrates a CI/CD pipeline using **Git** for source control, **Jenkins** for automation, and **Docker** for containerized deployment of a Java web application (`.war` file).

---

## 🔁 Project Flow

```
GitHub (Code Commit)
   ⮕ Jenkins (Builds + Transfers WAR)
      ⮕ Docker Host (WAR deployed inside Docker container running Tomcat)
```

---

## 🐳 Docker Host Setup (EC2 Instance 1)

### Step 1: Install Docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

### Step 2: Create Docker Management User
```bash
sudo su
useradd dockeradmin
passwd dockeradmin  # Set as 'dockeradmin'
usermod -aG docker dockeradmin
visudo
# Add below root:
dockeradmin ALL=(ALL) NOPASSWD: ALL
```

### Step 3: Create Docker Build Directory
```bash
mkdir /opt/docker
cd /opt/docker
```

### Step 4: Create Dockerfile
At `/opt/docker/Dockerfile`:
```Dockerfile
FROM tomcat:9.0.106-jre21
COPY ./webapp-1.0-SNAPSHOT.war /usr/local/tomcat/webapps
```

Optional (to deploy as root app):
```Dockerfile
FROM tomcat:9.0.106-jre21
COPY ./ROOT.war /usr/local/tomcat/webapps/
```

---

## 🧪 Jenkins Server Setup (EC2 Instance 2)

### Step 1: Install Java, Maven, and Jenkins
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

### Step 2: Configure Environment
Create `/etc/profile.d/maven.sh`:
```bash
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}
```

```bash
chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
```

---

## 🔧 Jenkins Configuration

### Step 1: Access Jenkins
- URL: `http://<Jenkins-EC2-Public-IP>:8080`
- Initial password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 2: Install Plugins
- Maven Integration Plugin
- Deploy to Container Plugin
- Publish Over SSH Plugin

### Step 3: Configure Tools
- Manage Jenkins → Global Tool Configuration

**JDK:**
- Name: `java21`
- Path: `/usr/lib/jvm/java-21-openjdk-amd64`

**Maven:**
- Name: `Maven3`
- Path: `/opt/maven`

---

## 🔐 SSH Access from Jenkins to Docker Host

### Step 1: Enable Password SSH on Docker Host
Edit:
```bash
sudo nano /etc/ssh/sshd_config
# Set PasswordAuthentication yes
```

If not effective:
```bash
cd /etc/ssh/sshd_config.d/
sudo vi 60-cloudimg-settings.conf
# Set PasswordAuthentication yes
```

Restart SSH:
```bash
sudo service ssh restart
```

---

## 🔌 Configure "Publish Over SSH"

**Manage Jenkins → Configure System → Publish over SSH**

- Name: `docker`
- Hostname: `<Docker EC2 Private IP>`
- Username: `dockeradmin`
- Password: `dockeradmin`
- Click "Test Configuration"

---

## 🛠️ Jenkins Pipeline Job Setup

**Type**: Maven Project

### A) Source Code Management
- Git URL: `<your GitHub repo>`
- Branch: `main`

### B) Build
- Root POM: `pom.xml`
- Goals:
```bash
clean install package
```

### C) Post-Build Action: Send WAR
- SSH Server: `docker`
- Source files: `webapp/target/*.war`
- Remove prefix: `webapp/target`
- Remote directory: `/opt/docker`
- Exec Command:
```bash
docker stop demo;
docker rm -f demo;
docker image rm -f demo;
cd /opt/docker;
docker build -t demo .
```

### D) Post-Build Action: Run Container
- SSH Server: `docker`
- Exec Command:
```bash
docker run -d --name demo -p 8090:8080 demo
```

📌 Ensure `/opt/docker` is owned by `dockeradmin`:
```bash
sudo chown -R dockeradmin:dockeradmin /opt/docker
```

---

## 🔍 Validation

### Step 1: Before Jenkins Build
```bash
docker images
docker ps -a
```

### Step 2: Trigger Jenkins Job

### Step 3: After Jenkins Build
Check:
```bash
docker images
docker ps -a
```
Should show:
- `demo` image
- `demo` container

### Step 4: Access App

Default WAR:
```
http://<Docker_Public_IP>:8090/webapp-1.0-SNAPSHOT/
```

If deployed as ROOT:
```
http://<Docker_Public_IP>:8090/
```

---

## 🧹 Optional: Deploy WAR as ROOT

```bash
mv webapp-1.0-SNAPSHOT.war ROOT.war
# Update Dockerfile to:
FROM tomcat:9.0.106-jre21
COPY ./ROOT.war /usr/local/tomcat/webapps/
# Then:
cd /opt/docker
docker build -t demo .
docker stop demo
docker rm demo
docker run -d --name demo -p 8090:8080 demo
```

---

## ✅ Result

- Jenkins builds `.war` from GitHub
- WAR transferred to Docker EC2
- Docker builds image and runs container
- App accessible at public IP

---

## 📎 References

- [Tomcat DockerHub](https://hub.docker.com/_/tomcat)
- [Jenkins Official Docs](https://www.jenkins.io/doc/)
- [Maven Guide](https://maven.apache.org/guides/)
- [Docker Getting Started](https://docs.docker.com/get-started/)

