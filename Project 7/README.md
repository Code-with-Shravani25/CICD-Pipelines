# Deploy NGINX App on Kubernetes using Minikube (Docker Driver)

This project demonstrates how to deploy an NGINX application on a local Kubernetes cluster using **Minikube** with the **Docker driver** on a Linux environment.

---

## 🛠️ Prerequisites

- Ubuntu/Debian-based Linux OS
- User with `sudo` privileges
- Internet access

---

## 📦 Installation Steps

### 1. Update System Packages

```bash
sudo apt update -y
sudo apt upgrade -y
```

---

### 2. Install Docker

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

---

### 3. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

### 4. Install Minikube

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

---

### 5. Configure Docker Permissions

```bash
sudo usermod -aG docker $USER
newgrp docker   # Apply group changes without logout
```

---

## 🚀 Start Minikube

```bash
sudo su
minikube start --driver=docker --force
```

> Verify Minikube is running:
```bash
kubectl get nodes
```

---

## 📄 Create Kubernetes Deployment and Service

### nginx-deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

### nginx-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

## 📥 Apply the Configurations

```bash
kubectl apply -f nginx-deployment.yml
kubectl apply -f nginx-service.yml
```

---

## 🌐 Access the Application

```bash
minikube service nginx-service --url
```

Example Output:

```text
http://192.168.49.2:30080
```

You can access the app using `curl` or by pasting the URL into your browser.

---

## ⚠️ Important Notes on Minikube + Docker Driver

- Minikube **does NOT expose NodePort services** to the public IP when using the Docker driver.
- It only binds to the internal Docker network (`127.0.0.1` or `bridge`).
- Accessing `http://<EC2-PUBLIC-IP>:30080` will not work directly.
- This is **not an EC2 limitation**, but a **Minikube + Docker driver design constraint**.

---

## ✅ Summary

This project covers:
- Setting up Docker, kubectl, and Minikube
- Running a local Kubernetes cluster
- Deploying and exposing an NGINX app using `Deployment` and `Service` manifests
