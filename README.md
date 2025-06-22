
# CI/CD Project: GitHub → Jenkins → Tomcat EC2 Deployment
This project demonstrates a basic CI/CD pipeline using GitHub for source control, Jenkins for building/testing/deployment, and Apache Tomcat running on an EC2 instance for hosting the application.


## Steps
## Step 1: Tomcat Setup on Web Server
Launch one EC2

Install Java

```bash
sudo apt update -y
sudo apt install openjdk-21-jdk -y
```
Install Tomcat

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
```bash
http://<Tomcat-EC2-Public-IP>:8080
```

Steps to Add a Role in tomcat-users.xml
```bash
sudo nano /opt/tomcat/conf/tomcat-users.xml
```
Add below lines
```bash
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="admin" password="admin123" roles="manager-gui,manager-script"/>
```
comment out <Valve> to disable IP-based access restriction 
```bash
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
```
example:
<Context antiResourceLocking="false" privileged="true" >
 <!-- <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1"/>
-->
</Context>


Restart Tomcat
```bash
cd /opt/tomcat/bin
./shutdown.sh
./startup.sh
```

Access Manager Web UI
```bash
http://<Tomcat-EC2-Public-IP>:8080/manager/html
```
Use the username and password you set (e.g., admin / admin123).

## Step 2: Jenkins Setup 
Launch one EC2 and allow 8080 port in Security Group.

Install Java
```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
java -version
```
Install Jenkins
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```
Install Maven
```bash
sudo su
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.10/binaries/apache-maven-3.9.10-bin.tar.gz
tar -xzvf apache-maven-3.9.10-bin.tar.gz
ln -s /opt/apache-maven-3.9.10 /opt/maven
```
Set Maven installation Directory Path
```bash
nano /etc/profile.d/maven.sh 
```
Add below lines
```bash
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}
```
Set permission and load environment variables
```bash
chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh 
```
To check Maven version
```bash
mvn -version
```
## Step 3: Login to Jenkins

Access Jenkins via http://<Jenkins-EC2-Public-IP>:8080

Retrieve the initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Install suggested plugins

Create first admin user

Tool Configuration

Go to: Manage Jenkins → Global Tool Configuration

Add Maven

Name: Maven3

Uncheck “Install automatically”

MAVEN_HOME: /opt/maven

Add Java

Name: java21

Path: /usr/lib/jvm/java-21-openjdk-amd64

📌 Ensure the path has both java and javac. If not, reinstall Java.

Install Required Jenkins Plugins

Go to: Manage Jenkins → Plugin Manager

Maven Integration Plugin

Deploy to Container Plugin

## Step4: Create Jenkins Maven Job

Create a Maven Project Job

Source Code Management:

Select Git

Add GitHub repository URL and Specify branch (e.g., master)

Example: https://github.com/Code-with-Shravani25/Project1.git

Build Configuration:

Path to pom.xml (if not in root)

Build goals:

```bash
clean install package
```

Post-Build Actions:

Select: Deploy war/ear to a container

WAR/EAR Files: Specify the path, e.g., target/your-app.war (webapp/target/*.war)

Container: Tomcat 8/9/10 remote

Add credentials (username/password that we use for tomcat manager app)
```bash
Tomcat URL: http://<Tomcat-EC2-Public-IP>:8080
```
Tomcat WAR Deployment Info

WAR files will be deployed to: /opt/tomcat/webapps/

## Access Application

```bash
http://<Tomcat-EC2-Public-IP>:8080/<your-app-name>/
```


