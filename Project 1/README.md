
# CI/CD Project: GitHub → Jenkins → Tomcat EC2 Deployment

This project demonstrates a basic CI/CD pipeline using GitHub for source control, Jenkins for building/testing/deployment, and Apache Tomcat running on an EC2 instance for hosting the application.

---

## 📋 Project Flow
```
GitHub (Code Commit) 
    ⮕ Jenkins (Build + Test) 
        ⮕ EC2 (Tomcat Server Deployment)
```

---

## 🖥️ Infrastructure Setup

### 🔹 Step 1: Launch 2 EC2 Instances
- **Jenkins Server**: Allow inbound port 8080 in the Security Group.
- **Tomcat Server**: Allow inbound port 8080 in the Security Group.

---

## 🔧 Step 2: Tomcat Setup on Web Server (EC2)

### ✅ Install Java
```bash
sudo apt update -y
sudo apt install openjdk-21-jdk -y
```

### ✅ Install Tomcat
```bash
sudo su
cd /opt
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.41/bin/apache-tomcat-10.1.41.tar.gz
tar -xvzf apache-tomcat-10.1.41.tar.gz
mv apache-tomcat-10.1.41 tomcat
cd tomcat/bin
./startup.sh
```

Access Tomcat:  
`http://<Tomcat-EC2-Public-IP>:8080`

### 📌 Note: Change default port in server.xml if needed

---

### Steps to Add a Role in tomcat-users.xml

1. Open file:
```bash
sudo nano /opt/tomcat/conf/tomcat-users.xml
```

2. Add roles:
```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="admin" password="admin123" roles="manager-gui,manager-script"/>
```

3. Save and exit

4. Modify:
```bash
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
```

Comment out:
```xml
<Context antiResourceLocking="false" privileged="true" >
 <!-- <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1"/>
-->
</Context>
```

5. Modify:
```bash
sudo nano /opt/tomcat/conf/Catalina/localhost/manager.xml
```

Replace or comment valve section:
```xml
<Context antiResourceLocking="false" privileged="true">
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
</Context>
```

---

## 🔄 Step 3: Restart Tomcat
```bash
cd /opt/tomcat/bin
./shutdown.sh
./startup.sh
```

---

## 🌐 Step 4: Access Manager Web UI

Visit:  
`http://<Tomcat-EC2-Public-IP>:8080/manager/html`  
Use: `admin / admin123`

---

## 🔧 Step 5: Jenkins Setup on EC2

### ✅ Install Java
```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
java -version
```

### ✅ Install Jenkins
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

### ✅ Install Maven
```bash
sudo su
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.tar.gz
tar -xzvf apache-maven-3.9.10-bin.tar.gz
ln -s /opt/apache-maven-3.9.10 /opt/maven

nano /etc/profile.d/maven.sh
# Add:
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}

chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
mvn -version
```

---

## ⚙️ Jenkins Configuration

### 🔑 Initial Setup
- Access: `http://<Jenkins-EC2-Public-IP>:8080`
- Get initial password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Install plugins, create admin user

### 🛠️ Tool Configuration
- Manage Jenkins → Global Tool Configuration

#### ➕ Add Maven
- Name: `Maven3`
- Uncheck Install automatically
- Path: `/opt/maven`

#### ➕ Add Java
- Name: `java21`
- Path: `/usr/lib/jvm/java-21-openjdk-amd64`

📌 Ensure both `java` and `javac` are available.

---

## 🔌 Install Required Jenkins Plugins

- Maven Integration Plugin
- Deploy to Container Plugin

---

## 🚀 Step 6: Create Jenkins Maven Job

### 🔨 Create a Maven Project Job

1. **Source Code Management**:
   - Git
   - GitHub repo URL
   - Branch (e.g., `main`)

2. **Build**:
   - Goals:
     ```bash
     clean install -DskipTests
     ```
   - Advanced → `MAVEN_OPTS`:
     ```bash
     -Dmaven.repo.local=/var/lib/jenkins/.m2/repository
     ```

3. **Repository Setup**:
```bash
sudo mkdir -p /var/lib/jenkins/.m2/repository
sudo chown -R jenkins:jenkins /var/lib/jenkins/.m2
```

4. **Post-Build**:
   - Deploy war/ear to container
   - WAR/EAR Files: `webapp/target/*.war`
   - Container: Tomcat 8/9/10 remote
     - Add Tomcat credentials (`admin/admin123`)
     - Tomcat URL: `http://<Tomcat-EC2-Public-IP>:8080`

---

## 📂 Tomcat WAR Deployment Info

- WARs go to: `/opt/tomcat/webapps/`
- Access at: `http://<Tomcat-EC2-Public-IP>:8080/<your-app-name>/`
- Or view in: `http://<Tomcat-EC2-Public-IP>:8080/manager/html`

---

## ✅ Final Outcome

- ✅ GitHub commit triggers Jenkins build
- ✅ Jenkins builds with Maven
- ✅ WAR deployed to Tomcat EC2
- ✅ App live on Tomcat public IP

---

## 📎 References

- [Tomcat Downloads](https://tomcat.apache.org/download-10.cgi)
- [Jenkins Installation Guide](https://www.jenkins.io/doc/)
