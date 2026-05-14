# Jenkins + SonarQube + Docker CI/CD Pipeline

A DevOps project that automates the build, code quality analysis, and deployment of a static website using Jenkins, SonarQube, and Docker across 3 AWS EC2 instances.

---

## What is it?

This project sets up a full CI/CD pipeline for a static digital marketing website (Onix Digital - TemplateMo 565).

Every time code is pushed to GitHub, Jenkins automatically:
1. Pulls the latest code
2. Runs a SonarQube code quality scan
3. Builds a Docker image and deploys it via nginx

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Jenkins | CI/CD automation |
| SonarQube | Code quality analysis |
| Docker + nginx | Containerized deployment |
| GitHub | Source code repository |
| AWS EC2 | Cloud infrastructure |

---

# Part 1 — Installation

## Architecture

```
EC2 #1 — Jenkins Server     (port 8080)
EC2 #2 — SonarQube Server   (port 9000)
EC2 #3 — Docker App Server  (port 80)
```

![EC2 - 3 Machines](Images/ec2-%203%20machines.png)

---

## Prerequisites

- 3 AWS EC2 instances (Ubuntu 22.04+)
- t2.medium for Jenkins and SonarQube, t2.micro for Docker
- GitHub account with this repo forked

---

## EC2 #1 — Jenkins

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | gpg --dearmor | sudo tee /usr/share/keyrings/jenkins-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list

sudo apt update && sudo apt install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Access at: `http://<EC2-1-IP>:8080`

Get initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

![Jenkins Dashboard](Images/Jenkins%20dashboard.png)

---

## EC2 #2 — SonarQube

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk unzip wget

sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
unzip sonarqube-10.4.1.88267.zip
sudo mv sonarqube-10.4.1.88267 /opt/sonarqube
sudo chown -R ubuntu:ubuntu /opt/sonarqube

/opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

Access at: `http://<EC2-2-IP>:9000`
Default login: `admin / admin`

![SonarQube Server URL](Images/sonarqube%20server-%20url.png)
![SonarQube Dashboard](Images/sonarqube%20-%20dashboard.png)

---

## EC2 #3 — Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo chmod 666 /var/run/docker.sock
sudo usermod -aG docker ubuntu
mkdir -p /home/ubuntu/app
```

---

## Jenkins Configuration

### Plugins to install
- SonarQube Scanner
- SSH Agent

### SonarQube Integration
1. Generate token in SonarQube → My Account → Security → Generate Token
2. Manage Jenkins → Credentials → Add → Secret text → paste token
3. Manage Jenkins → System → SonarQube servers → add URL `http://<EC2-2-IP>:9000`
4. Manage Jenkins → Tools → SonarQube Scanner → add with name `sonar-scanner`

### SSH Setup (Jenkins → Docker server)
```bash
# On EC2 #1
sudo -u jenkins ssh-keygen -t rsa -b 2048 -f /var/lib/jenkins/.ssh/id_rsa -N ""
sudo cat /var/lib/jenkins/.ssh/id_rsa.pub
```

Copy the public key to EC2 #3:
```bash
echo "<jenkins-public-key>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Add private key in Jenkins → Credentials → SSH Username with private key → ID: `docker-server`

![Jenkins Credentials](Images/jenkins-%20creditt.png)

---

## Security Groups

| EC2 | Port | Source |
|---|---|---|
| Jenkins | 22, 8080 | Your IP |
| SonarQube | 22, 9000 | Your IP + Jenkins IP |
| Docker | 22 | Your IP + Jenkins IP |
| Docker | 80 | 0.0.0.0/0 |

![SG - Inbound Rules](Images/SG%20-%20inbound.png)
![SG - Outbound Rules](Images/SG%20-%20outbound.png)

---

## Storage Requirements

| EC2 | Minimum |
|---|---|
| Jenkins | 20 GB |
| SonarQube | 30 GB |
| Docker | 15 GB |

---

# Part 2 — Working

## How it Works

```
Push code to GitHub
        ↓
Jenkins (EC2 #1) pulls the code
        ↓
SonarQube (EC2 #2) scans for bugs and code issues
        ↓
Jenkins SSHs into EC2 #3
        ↓
Docker builds the nginx image
        ↓
Container runs → site is live on port 80
```

---

## Pipeline Script

```groovy
pipeline {
    agent any
    stages {
        stage('Clone') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-repo.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "${tool 'sonar-scanner'}/bin/sonar-scanner"
                }
            }
        }
        stage('Deploy to Docker Server') {
            steps {
                sshagent(['docker-server']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@<EC2-3-IP> "
                            docker stop my-website || true &&
                            docker rm my-website || true &&
                            docker rmi my-website || true
                        "
                        scp -r ./* ubuntu@<EC2-3-IP>:/home/ubuntu/app/
                        ssh ubuntu@<EC2-3-IP> "
                            cd /home/ubuntu/app &&
                            docker build -t my-website . &&
                            docker run -d -p 80:80 --name my-website my-website
                        "
                    '''
                }
            }
        }
    }
}
```

---

## Pipeline Results

![Jenkins - My Website Job](Images/jenkins-my%20website.png)
![Jenkins - My Website Status](Images/jenkins-%20my%20website%20-%20status.png)
![Docker Status](Images/docker%20status.png)
![Docker Website](Images/docker%20webiste.png)
