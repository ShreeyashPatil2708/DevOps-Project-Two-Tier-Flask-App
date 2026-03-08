# Two-Tier Flask App with Docker & Jenkins CI/CD on AWS

**Author:** Shreeyash Patil
**Institution:** Pimpri Chinchwad College of Engineering (PCCOE), Pune
**Date:** March 2026

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Step 1: AWS EC2 Instance Setup](#4-step-1-aws-ec2-instance-setup)
5. [Step 2: Install Dependencies](#5-step-2-install-dependencies)
6. [Step 3: Jenkins Installation and Setup](#6-step-3-jenkins-installation-and-setup)
7. [Step 4: Repository File Structure](#7-step-4-repository-file-structure)
8. [Step 5: Jenkins Pipeline Configuration](#8-step-5-jenkins-pipeline-configuration)
9. [Step 6: GitHub Webhook Setup](#9-step-6-github-webhook-setup)
10. [Conclusion](#10-conclusion)

---

## 1. Project Overview

I built this to get hands-on with the DevOps tools I kept seeing in job descriptions — Docker, Jenkins, CI/CD pipelines. The app itself is simple: a Flask + MySQL message board running on AWS EC2. The interesting part is everything around it.

Push code to GitHub → Jenkins picks it up automatically → rebuilds the Docker image → redeploys the containers. No SSH-ing into the server, no manual docker commands. Just push and it's live.

The app lets users type and submit messages, which get stored in MySQL and shown on the homepage. Nothing fancy, but it's enough to have a real two-tier architecture to containerize and wire together.

---

## 2. Architecture

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same AWS EC2)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

---

## 3. Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend & App Logic | Flask (Python) |
| Database | MySQL |
| Containerization | Docker & Docker Compose |
| CI/CD | Jenkins |
| Version Control | GitHub |
| Cloud Infrastructure | AWS EC2 (Ubuntu 22.04, t3.small) |

---

## 4. Step 1: AWS EC2 Instance Setup

**Launch EC2 Instance:**
- AMI: Ubuntu 22.04 LTS
- Instance type: t3.small (2GB RAM)
- Storage: 15GB
- Key pair: create and save your `.pem` file

**Configure Security Group — open these inbound ports:**

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH access |
| 8080 | TCP | Jenkins dashboard |
| 5000 | TCP | Flask application |

**Security Note:**
For this learning/demo project, all ports are open to `0.0.0.0/0` (anywhere) for simplicity. In a production environment, best practices would be:
- **Port 22 (SSH)** — restrict to your own IP only
- **Port 8080 (Jenkins)** — restrict to your IP or put Jenkins behind a VPN
- **Port 5000 (Flask)** — `0.0.0.0/0` is fine since it's a public web app

**Connect to EC2:**
```bash
ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>
```

---

## 5. Step 2: Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu

# Install Docker Compose
sudo apt install docker-compose -y

# Install Java (required for Jenkins)
sudo apt install openjdk-17-jdk -y
```

---

## 6. Step 3: Jenkins Installation and Setup

```bash
# Add Jenkins GPG key
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7198F4B714ABFC68

# Add Jenkins repo
echo "deb https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list

# Install Jenkins
sudo apt update && sudo apt install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Add jenkins user to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Access Jenkins at `http://<ec2-public-ip>:8080` and complete the setup wizard.

Get the initial admin password with:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

> **Note:** The standard curl/wget method for adding the Jenkins GPG key fails on Ubuntu 22.04 with a `NO_PUBKEY` error. The fix that actually works is fetching the key directly from Ubuntu's keyserver using the key ID from the error message.

---

## 7. Step 4: Repository File Structure

```
DevOps-Project-Two-Tier-Flask-App/
├── app.py                  # Flask application
├── Dockerfile              # Docker image definition for Flask
├── docker-compose.yml      # Orchestrates Flask + MySQL containers
├── Jenkinsfile             # CI/CD pipeline definition
├── requirement.txt         # Python dependencies
├── message.sql             # SQL to create messages table
└── templates/
    └── index.html          # Frontend UI
```

**Dockerfile:**
```dockerfile
FROM python:3.9-slim
WORKDIR /app
RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*
COPY requirement.txt .
RUN pip install --no-cache-dir -r requirement.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

**docker-compose.yml:**
```yaml
version: "3.8"
services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "devops"
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - two-tier-nt
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot","-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask-app:
    container_name: two-tier-app
    build:
      context: .
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=devops
    networks:
      - two-tier-nt
    depends_on:
      mysql:
        condition: service_healthy

volumes:
  mysql_data:

networks:
  two-tier-nt:
```

**Jenkinsfile:**
```groovy
pipeline {
    agent any
    stages {
        stage('Clone repo') {
            steps {
                git branch: 'main', url: 'https://github.com/ShreeyashPatil2708/DevOps-Project-Two-Tier-Flask-App.git'
            }
        }
        stage('Build image') {
            steps {
                sh 'docker build -t flask-app .'
            }
        }
        stage('Deploy with docker compose') {
            steps {
                sh 'docker-compose down || true'
                sh 'docker-compose up -d --build'
            }
        }
    }
}
```

---

## 8. Step 5: Jenkins Pipeline Configuration

1. From Jenkins dashboard → **New Item** → name it → select **Pipeline** → OK
2. In **Triggers** → check **"GitHub hook trigger for GITScm polling"**
3. In **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/ShreeyashPatil2708/DevOps-Project-Two-Tier-Flask-App`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
4. Click **Save** → click **Build Now** for the first manual run

---

## 9. Step 6: GitHub Webhook Setup

1. Go to your GitHub repo → **Settings** → **Webhooks** → **Add webhook**
2. Fill in:
   - **Payload URL:** `http://<ec2-public-ip>:8080/github-webhook/`
   - **Content type:** `application/json`
   - **SSL verification:** Disable (Jenkins runs on plain HTTP, not HTTPS, so SSL verification will fail)
3. Click **Add webhook**

This is what makes it a real CI/CD pipeline. Without the webhook you're still clicking "Build Now" manually every time.

---

## 10. Conclusion

The full flow once everything is wired up:

```
git push → GitHub webhook → Jenkins triggered → Docker image built → Containers deployed → App live
```

The part I found most satisfying was pushing a code change and watching Jenkins kick off on its own. That's the whole point — you stop thinking about deployment as a separate manual task.

One thing I'd do differently in a real setup: restrict port 22 and 8080 to specific IPs instead of leaving them open to `0.0.0.0/0`. Fine for a learning project, not for production.

---

## About

3rd year Computer Engineering student at PCCOE, Pune. Built this project to learn Docker and Jenkins properly, not just read about them.

**GitHub:** [ShreeyashPatil2708](https://github.com/ShreeyashPatil2708)
