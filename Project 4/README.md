
# 🚀 CI/CD Project: Git → Jenkins → Ansible → Docker Hub → Docker Host Deployment

This project sets up a complete CI/CD pipeline where Jenkins builds and tests a Java web application, Ansible pushes the image to Docker Hub, and a Docker host pulls and deploys the image from Docker Hub.

---

## 📦 Project Flow

```
GitHub (Source Code)
   ⮕ Jenkins (Build and Test)
      ⮕ Ansible Server (Docker Build + Push to Docker Hub)
         ⮕ Docker Host (Pull from Docker Hub + Deploy Container)
```

---

## 🔧 Prerequisites

- Jenkins Server EC2 (with Maven)
- Ansible Server EC2 (with Docker)
- Docker Host EC2 (with Docker)
- Docker Hub account (e.g., `shravani2001`)
- GitHub repo (with `Dockerfile` and `.war` source)

---

## 🧱 Jenkins & Maven Setup (on EC2)

Install Java, Jenkins, and Maven, and configure environment variables and paths. Also configure the "Publish Over SSH" plugin for pushing files and commands to Ansible.

---

## 🐳 Docker Host EC2 Setup

Install Docker, create user `ansadmin`, add to Docker group, set permissions, and make `/opt/docker` accessible by `ansadmin`.

---

## ⚙️ Ansible Server Setup

Install Ansible and Docker, create `ansadmin` user, allow password SSH, and configure Docker Hub login.

---

## 🏗️ Jenkins Build Configuration

- **Job Type**: Maven
- **GitHub Repository**: https://github.com/Code-with-Shravani25/Project2.git
- **Goals**: `clean install package`
- **Send WAR + Dockerfile to Ansible Server**
- **Post-Build Exec**:
```bash
cd /opt/docker
docker build -t demo .
docker tag demo shravani2001/demo
docker push shravani2001/demo
docker rmi demo shravani2001/demo
```

---

## 📜 Ansible Playbook

Location: `/etc/ansible/playbook.yml`

```yaml
---
- hosts: webserver
  become: true
  tasks:
    - name: Stop previous container
      shell: docker stop demo || true

    - name: Remove old container
      shell: docker rm -f demo || true

    - name: Remove old image
      shell: docker rmi -f shravani2001/demo || true

    - name: Pull image from Docker Hub
      shell: docker pull shravani2001/demo

    - name: Run new container
      shell: docker run -d --name demo -p 8090:8080 shravani2001/demo
```

---

## 📂 Ansible Inventory

File: `/etc/ansible/hosts.ini`

```ini
[webserver]
<docker-host-private-ip> ansible_user=ansadmin ansible_ssh_pass=ansadmin
```

---

## 🛠️ Jenkins Post Build Action (Trigger Ansible)

In Post-build:
```bash
cd /etc/ansible
ansible-playbook -i hosts.ini playbook.yml
```

---

## 🔍 Validation & Testing

- Trigger Jenkins job
- Check image uploaded to Docker Hub
- Check container running on Docker Host:
```bash
docker ps -a
```
- Access Application:
```
http://<Docker_Public_IP>:8090/webapp-1.0-SNAPSHOT/
```

---

## ✅ Outcome

- Jenkins builds WAR and Dockerfile
- Ansible builds image, pushes to Docker Hub
- Docker Host pulls and runs container
- Application runs inside Tomcat container

---

## 📎 References

- [Docker Hub](https://hub.docker.com/)
- [Jenkins Official Docs](https://www.jenkins.io/doc/)
- [Ansible Docs](https://docs.ansible.com/)
- [Apache Tomcat Docker Image](https://hub.docker.com/_/tomcat)

---

## 📌 Notes

- Ensure SSH access between Jenkins and Ansible
- `ansadmin` must have sudo access and be in the Docker group
- Dockerfile and WAR must be present in GitHub repo
- Permissions of `/opt/docker` and Ansible files must be set correctly
- Use `sshpass` if password authentication is used in Jenkins-Ansible communication

