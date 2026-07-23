# CI/CD Pipeline for Tomcat Application using Jenkins

## Project Overview

This project demonstrates a complete CI/CD pipeline for deploying a Java Tomcat web application using the following tools:

1. GitHub - Source Code Management
2. Maven - Build Tool
3. Jenkins - CI/CD Automation Tool
4. SonarQube - Code Quality Analysis
5. Trivy - Docker Image Vulnerability Scanner
6. Docker - Containerization Platform

---

# Architecture

GitHub → Jenkins → Maven Build → SonarQube Scan → Docker Build → Trivy Scan → DockerHub → Docker Container

---

# Step 1 : Jenkins Server Setup on Ubuntu Linux VM

## 1. Create Ubuntu VM

* Create Ubuntu Linux VM
* Enable inbound ports:

  * 8080 (Jenkins)
  * 9000 (SonarQube)
  * 8010 (Application)

---

## 2. Connect to VM

```bash
ssh username@server-ip
```

---

## 3. Install Java 21

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y

java -version
```

---

## 4. Install Jenkins

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update

sudo apt install jenkins -y
```

---

## 5. Start Jenkins Service

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Verify:

```bash
sudo systemctl status jenkins
```

---

## 6. Access Jenkins

```text
http://PUBLIC-IP:8080
```

---

## 7. Get Jenkins Initial Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 8. Create Jenkins Admin User

* Create admin username/password
* Install suggested plugins

---

# Jenkins Pipeline Types

## Declarative Pipeline

* Simple and structured syntax
* Recommended for beginners
* Uses predefined stages and steps

## Scripted Pipeline

* Written using Groovy scripting
* More flexible
* Suitable for advanced workflows

---

# Step 2 : Configure Maven in Jenkins

Go to:

```text
Manage Jenkins → Tools
```

Add Maven:

```text
Name: maven3
```

---

# Step 3 : Install Docker on Jenkins Server

```bash
curl -fsSL https://get.docker.com | sudo /bin/bash

sudo usermod -aG docker jenkins

sudo systemctl restart jenkins

docker version
```

---

# Step 4 : Configure GitHub Repository

## Create GitHub Repository

Example repository:

```text
https://github.com/srinidks/cicd-tomcat-2026.git
```

---

## Push Source Code to GitHub

```bash
git init

git remote add origin https://github.com/srinidks/cicd-tomcat-2026.git

git remote -v

git branch -M main

git status

git add .

git commit -m "Source code added"

git push -u origin main
```

---

# Step 5 : Install SonarQube Server

Run SonarQube using Docker:

```bash
docker run -d \
--name sonarqube \
-p 9000:9000 \
sonarqube:latest
```

Access SonarQube:

```text
http://PUBLIC-IP:9000
```

Default credentials:

```text
Username: admin
Password: admin
```

---

# Step 6 : Install Trivy, Image and File System Scan
# 1 Install Trivy:
```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

# 2 Verify:

```bash
trivy --version
```
---
```text
* Scan the image and file Repository
```bash
trivy fs .
trivy image httpd:latest
```
* Fail pipeline on critical/high vulns
```bash
trivy image --exit-code 1 --severity HIGH,CRITICAL httpd:latest

* JSON output (good for parsing/archiving)
```bash
trivy image --format json -o trivy-image-report.json tomcat-app
```
* Ignore unfixed vulnerabilities (no patch available yet)
```bash
trivy image --ignore-unfixed tomcat-app
```
---
# step 7: Install the plugins:
```text
Manage Jenkins Plugins -> available plugins

* git
* ssh-agent
* Maven Integration
* docker
* SonarQube Scanner
* Pipeline: Stage View

# Step 7 : Configure Credentials in Jenkins

Go to:

```text
Manage Jenkins → Credentials
```

Add the following credentials.

---

## GitHub Credential

* Username
* GitHub Personal Access Token

Credential ID:

```text
GitHub
```

---

## DockerHub Credential

* DockerHub Username
* DockerHub Access Token

Credential ID:

```text
DokckerHub
```

---

## SonarQube Token

* Kind: Secret Text
* Add SonarQube token

Credential ID:

```text
sonar-token
```

---

# Step 8 : Configure SonarQube in Jenkins

Go to:

```text
Manage Jenkins → System
```

Under SonarQube Servers:

| Field | Value                 |
| ----- | --------------------- |
| Name  | sonarqube             |
| URL   | http://PUBLIC-IP:9000 |
| Token | sonar-token           |

---

# Step 9 : Configure GitHub Webhook Trigger

## Install Plugins

Go to:

```text
Manage Jenkins → Plugins
```

Install:

* GitHub Integration Plugin
* Git Plugin

---

## Configure Jenkins Job

Open Jenkins Pipeline Job:

```text
Job → Configure
```

Under:

```text
Build Triggers
```

Enable:

```text
GitHub hook trigger for GITScm polling
```

Save the job.

---

## Configure GitHub Webhook

Open GitHub Repository:

```text
Repository → Settings → Webhooks
```

Click:

```text
Add webhook
```

Payload URL:

```text
http://JENKINS-PUBLIC-IP:8080/github-webhook/
```

Example:

```text
http://198.18.8.42:8080/github-webhook/
```

Content type:

```text
application/json
```

Select:

```text
Just the push event
```

Click:

```text
Add webhook
```

---

# Step 10 : Create Jenkins Pipeline Job

## Pipeline Stages

### Stage 1 : Checkout Code from GitHub

Pulls source code from GitHub repository.

---

### Stage 2 : Build Application using Maven

Builds WAR package.

---

### Stage 3 : SonarQube Code Quality Check

Checks:

* Bugs
* Vulnerabilities
* Code smells

---

### Stage 4 : Create Docker Image

Creates Docker image.

---

### Stage 5 : Scan Docker Image using Trivy

Scans HIGH and CRITICAL vulnerabilities.

---

### Stage 6 : Push Docker Image to DockerHub

Pushes image to DockerHub repository.

---

### Stage 7 : Create Docker Container

Runs Docker container automatically.

---

# Step 11 : Jenkins Pipeline Script

```groovy
pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'java21'
    }

    environment {
        IMAGE_NAME = "srinidks/tomcat-app"
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout code from GitHub') {
            steps {
                git branch: 'main',
                credentialsId: 'GitHub',
                url: 'https://github.com/srinidks/cicd-tomcat-2026.git'
            }
        }

        stage('Build the Application') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Code Quality check using SonarQube') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Create Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Scan the Docker Image') {
            steps {
                sh 'trivy image --severity HIGH,CRITICAL --scanners vuln $IMAGE_NAME:$IMAGE_TAG'
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                withDockerRegistry(credentialsId: 'DokckerHub', url: 'https://index.docker.io/v1/') {
                    sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
                }
            }
        }

        stage('Create the Docker Container') {
            steps {
                sh '''
                docker rm -f tomcatapp || true

                docker run -d \
                --name tomcatapp \
                -p 8010:8080 \
                $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }
    }

    post {

        success {
            echo 'Pipeline executed successfully'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}
```

---

# Step 12 : Trigger Jenkins Job

* Click Build Now
* Verify all stages complete successfully

---

# Step 13 : Verify GitHub Webhook

Push code changes:

```bash
git add .

git commit -m "Webhook testing"

git push origin main
```

Verify Jenkins job triggers automatically.

---

# Step 14 : Access Application

```text
http://PUBLIC-IP:8010/tomcat-app
```

---

# Step 15 : Verify Docker Container

```bash
docker ps
```

---

# Step 16 : Verify Docker Images

```bash
docker images
```

---

# Step 17 : Verify SonarQube Dashboard

```text
http://PUBLIC-IP:9000
```

---

# Step 18 : Verify Trivy Scan

```bash
trivy image srinidks/tomcat-app:1
```

---

# Final CI/CD Flow

1. Developer pushes code to GitHub
2. GitHub webhook triggers Jenkins
3. Jenkins pulls source code
4. Maven builds WAR package
5. SonarQube analyzes code quality
6. Docker image is created
7. Trivy scans image vulnerabilities
8. Docker image pushed to DockerHub
9. Docker container deployed automatically

---

# Author

Srinivasalu
