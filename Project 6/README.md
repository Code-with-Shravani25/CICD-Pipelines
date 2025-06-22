# 📘 README: GitHub → Jenkins Webhook Integration

This guide walks through setting up a GitHub webhook to automatically trigger Jenkins jobs when code is pushed to a repository.

---

## 🚀 Project Flow

**GitHub (Push Code)**  
⮕ **Webhook triggers Jenkins job**  
⮕ **Jenkins pulls and builds code**

---

## 🖥️ 1. Launch EC2 & Install Jenkins

```bash
sudo apt update -y
sudo apt install openjdk-21-jdk -y

# Jenkins installation
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

Access Jenkins at:  
`http://<EC2-Public-IP>:8080`

---

## 🔧 2. Configure Jenkins Project

1. Go to: Jenkins Dashboard → Your Job → **Configure**
2. Under **Build Triggers**, enable:
   - ✅ **GitHub hook trigger for GITScm polling**
3. Under **Source Code Management**:
   - Select **Git**
   - Add your GitHub repo **(SSH or HTTPS)**
   - Add **credentials** if private

---

## 🔑 3. Get Jenkins Webhook URL

Webhook format:  
```
http://<JENKINS_PUBLIC_IP>:8080/github-webhook/
```

🔁 Jenkins listens for incoming GitHub payloads at `/github-webhook/`.

Example:  
```
http://13.235.120.8:8080/github-webhook/
```

---

## 🌐 4. Create Webhook in GitHub

1. Navigate to your **GitHub Repository**
2. Go to **Settings → Webhooks → Add Webhook**
3. Fill the form:

   - **Payload URL**:  
     `http://<Jenkins-IP>:8080/github-webhook/`
   - **Content Type**:  
     `application/json`
   - **Secret**: (Optional)
   - **Event**:  
     ✅ Just the **push event** or select custom events

4. Click **Add Webhook**

---

## 🔄 5. Test the Webhook

1. Make a code change and **push** to the GitHub repo.
2. GitHub sends a payload to Jenkins.
3. Jenkins job should **automatically trigger**.

You can view the **delivery status** in GitHub:  
**Repo → Settings → Webhooks → Recent Deliveries**

---

## ✅ Final Outcome

- Every **push to GitHub** triggers the **Jenkins build**
- **No manual job triggering** needed
- Great for **CI automation**

---

## 📎 References

- [GitHub Webhooks Documentation](https://docs.github.com/en/developers/webhooks-and-events/webhooks)
- [Jenkins GitHub Plugin](https://plugins.jenkins.io/github/)
