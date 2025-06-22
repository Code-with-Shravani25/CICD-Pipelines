
# CI/CD Pipeline: GitHub → Jenkins → Ansible → Tomcat EC2 Deployment

This project demonstrates a complete CI/CD pipeline using **GitHub** for source control, **Jenkins** for build automation, **Ansible** for deployment orchestration, and **Apache Tomcat** running on an EC2 instance for hosting the application.

---

## 📌 Project Flow

```
GitHub (Code Commit) 
   ⮕ Jenkins (Build + Test + Artifact Transfer)
      ⮕ Ansible Server (Triggers Deployment Playbook)
         ⮕ Tomcat Web Server (WAR Deployment)
```

---

## 🖥️ Infrastructure Setup (Tomcat EC2)

### Step 1: Install Java & Tomcat
```bash
sudo apt update -y
sudo apt install openjdk-21-jdk -y
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

---

### Step 2: Enable Tomcat Manager Access

Edit `tomcat-users.xml` and add roles:
```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="admin" password="admin123" roles="manager-gui,manager-script"/>
```

Modify access control files:
```bash
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
# Comment out the <Valve> section

sudo nano /opt/tomcat/conf/Catalina/localhost/manager.xml
# Comment or adjust valve as needed
```

Restart Tomcat:
```bash
cd /opt/tomcat/bin
./shutdown.sh
./startup.sh
```

---

## ⚙️ Jenkins Setup (Jenkins EC2)

### Step 1: Install Java, Jenkins & Maven
```bash
sudo apt update
sudo apt install openjdk-21-jdk -y

# Jenkins Installation
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

# Maven Installation
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.tar.gz
tar -xzvf apache-maven-3.9.10-bin.tar.gz
ln -s /opt/apache-maven-3.9.10 /opt/maven
```

---

### Step 2: Configure Environment
Create `/etc/profile.d/maven.sh` and add:
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

## 🔧 Jenkins UI Configuration

Access Jenkins:  
`http://<Jenkins-EC2-Public-IP>:8080`

Unlock Jenkins:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Create first admin user

---

### Global Tool Configuration

**Manage Jenkins → Global Tool Configuration:**

- **Add Maven:**
  - Name: `Maven3`
  - Install automatically: **Uncheck**
  - Path: `/opt/maven`

- **Add Java:**
  - Name: `java21`
  - Path: `/usr/lib/jvm/java-21-openjdk-amd64`

---

### 🔌 Required Jenkins Plugins

- Maven Integration Plugin  
- Deploy to Container Plugin  
- Publish Over SSH Plugin  

---

## 🔄 Ansible Server Setup

### Ansible Installation:
```bash
sudo apt install ansible -y
```

---

### Step 1: User Setup on Ansible & Tomcat Servers

On both EC2s:
```bash
sudo su -
useradd ansadmin
passwd ansadmin  # Set password: ansadmin
visudo
# Add under root:
ansadmin ALL=(ALL) NOPASSWD: ALL
```

---

### Step 2: SSH Access from Ansible → Tomcat

On Ansible server (as `ansadmin`):
```bash
ssh-keygen
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Copy public key to Tomcat:
```bash
vi ~/.ssh/authorized_keys  # Paste the Ansible public key
chmod 600 ~/.ssh/authorized_keys
```

---

### Step 3: Ansible Inventory & Playbook

**Inventory File:** `/etc/ansible/hosts`
```
[webserver]
<tomcat-ec2-private-ip>
```

**Playbook:** `/etc/ansible/playbook.yml`
```yaml
---
- name: Copy WAR to Tomcat Server
  hosts: webserver
  become: true
  tasks:
    - name: Deploy WAR
      copy:
        src: /home/ansadmin/webapp/target/webapp-1.0-SNAPSHOT.war
        dest: /opt/tomcat/webapps
```

---

## 🔐 Jenkins SSH to Ansible Configuration

### Step 1: Jenkins Key Setup

On Jenkins EC2:
```bash
sudo su - jenkins
ssh-keygen -t rsa -b 2048
cat ~/.ssh/id_rsa.pub
```

---

### Step 2: Add Jenkins Key to Ansible

On Ansible EC2 (`ansadmin`):
```bash
vi ~/.ssh/authorized_keys  # Paste Jenkins public key
chmod 600 ~/.ssh/authorized_keys
chown -R ansadmin:ansadmin ~/.ssh
```

---

### Step 3: Jenkins "Publish over SSH"

Go to:
**Manage Jenkins → Configure System → Publish over SSH**

- **Name**: `ansible`
- **Host**: `<Ansible private IP>`
- **Username**: `ansadmin`
- **Use Key**: ✅ Check
- **Key**: Paste content of Jenkins private key (`~/.ssh/id_rsa`)
- **Test Connection**

---

## 🧪 Jenkins Maven Job Configuration

**Source Code Management**
- Git
- Repo URL: `<your GitHub repo>`
- Branch: `main`

**Build**
- Goals:
  ```bash
  clean install -DskipTests
  ```

- Advanced → `MAVEN_OPTS`:
  ```bash
  -Dmaven.repo.local=/var/lib/jenkins/.m2/repository
  ```

**Post-Build Actions**
- **Send build artifacts over SSH**
  - **Source files**: `webapp/target/*.war`
  - **Remote directory**: `/home/ansadmin/webapp/target/`

- **Exec command**:
  ```bash
  ansible-playbook /etc/ansible/playbook.yml
  ```

---

## ✅ Outcome

- Code pushed to GitHub triggers Jenkins build.
- Jenkins compiles code and generates a WAR file.
- WAR is copied to the Ansible server.
- Ansible deploys the WAR to Tomcat on a separate EC2.
- Application is available at:  
  `http://<Tomcat-EC2-Public-IP>:8080/<app-name>`

---

## 📎 References

- [Tomcat Downloads](https://tomcat.apache.org/download-10.cgi)
- [Jenkins Official Docs](https://www.jenkins.io/doc/)
- [Ansible Documentation](https://docs.ansible.com/)

---

## 📌 Notes

- Ensure all file permissions are correct.
- Ownership of `/etc/ansible` and WAR target paths should be with `ansadmin`.
- Ensure all SSH keys are exchanged correctly for Ansible and Jenkins communication.

