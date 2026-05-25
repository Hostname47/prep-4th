# Jenkins

## What is Jenkins?

Jenkins is an open-source automation server mainly used for Continuous Integration (CI) and Continuous Delivery (CD). It allows automation of software development tasks such as running tests, deploying applications, and much more.

Jenkins is extensible through plugins and supports running tasks across different environments such as Docker, Kubernetes, or local machines.

- Open-source automation server for CI/CD (Continuous Integration / Continuous Delivery)
- Automates:
  - Build
  - Test
  - Deploy
  - Code analysis
- Highly extensible via plugins
- Supports Docker, Kubernetes, VMs, local machines

---

## 2. CI/CD Concept

- CI (Continuous Integration): frequent code integration + automated tests
- CD (Continuous Delivery/Deployment): automated release/deployment pipeline
- Goal: fast + reliable software delivery

---

## 3. Jenkins Installation (Ubuntu/Debian)

- Install Java (required)
- Add Jenkins repository
- Install Jenkins package
- Start Jenkins service

Main service commands:

- systemctl start jenkins
- systemctl enable jenkins
- systemctl status jenkins

---

## 4. Access Jenkins

- Local: `http://localhost:8080`
- Remote: `http://server-ip:8080`

Initial password:

- /var/lib/jenkins/secrets/initialAdminPassword

---

## 5. Jenkins Setup (First Login)

- Install suggested plugins OR custom plugins
- Recommended plugins:
  - Docker Pipeline
  - Docker plugin
  - Blue Ocean
  - Pipeline Stage View

- Create first admin user (username/password/email)

---

## 6. Jenkins Jobs Types

- Freestyle project (basic tasks)
- Pipeline project (recommended for CI/CD)
- Multibranch pipeline (auto detects branches)

---

## 7. Jenkins Pipeline (Core Concept)

Pipeline = code-based CI/CD process (Jenkinsfile)

Two types:

- Scripted pipeline (advanced)
- Declarative pipeline (recommended, structured)

Main structure:

- agent → where it runs (docker, node, etc.)
- stages → steps of pipeline
- steps → commands executed

Typical stages:

- Build
- Test
- SonarQube Analysis
- Package
- Deploy

---

## 8. Jenkinsfile from Git (SCM)

- Store Jenkinsfile in Git repository
- In Jenkins job:
  - Select “Pipeline from SCM”
  - Add Git URL
  - Add credentials (Git token)
  - Select branch
  - Select Jenkinsfile path

---

## 9. Credentials (VERY IMPORTANT)

Used to store secrets securely in Jenkins.

Types:

- Username + Password → DockerHub login
- Secret Text → API tokens (SonarQube, Git PAT)
- SSH Key → Git access

Usage:

- referenced in pipeline using credentialsId

---

## 10. GitHub PAT (Personal Access Token)

Used instead of password for GitHub authentication

Steps:

- GitHub → Settings → Developer settings → Personal Access Tokens
- Generate token
- Select scopes:
  - repo (full access to repositories)
- Use token in Jenkins credentials (NOT password)

---

## 11. Docker with Jenkins

Important requirement:

- Docker installed on Jenkins machine
- Jenkins user must be in docker group:
  - usermod -aG docker jenkins

Pipeline usage:

- build image
- tag image
- push to DockerHub
- run container

DockerHub credentials:

- Username + password stored in Jenkins credentials

---

## 12. SonarQube (Code Quality Analysis)

Purpose:

- Static code analysis
- Detect bugs, vulnerabilities, code smells

Run SonarQube:

- docker run -p 9000:9000 sonarqube

Access:

- `http://localhost:9000`
- default login:
  - admin / admin

Setup:

- Create project (Local project)
- Define:
  - Project name
  - Project key
  - Branch
- Generate token
- Choose Maven/Scanner instructions

Jenkins integration:

- store token as credential (Secret Text)
- use stage “SonarQube Analysis”
- run before build or after build

---

## 13. Jenkins + Docker + Sonar Pipeline Flow

Typical flow:

1. Checkout code (Git)
2. Build (Maven / Gradle)
3. Test
4. SonarQube analysis
5. Package artifact
6. Build Docker image
7. Push image to DockerHub
8. Deploy container

---

## 14. Docker Agent in Pipeline

Used to run pipeline inside container:

- docker image contains tools (maven + java + docker)
- mounts:
  - ~/.m2 (maven cache)
  - /var/run/docker.sock (docker access)

---

## 15. Common Pipeline Concepts

- environment → variables (URLs, tokens)
- withCredentials → secure access to secrets
- post → actions after pipeline (success/failure)
- triggers → schedule (cron)

---

## 16. Security Best Practices

- Never hardcode passwords/tokens
- Always use Jenkins credentials
- Use least privilege access
- Rotate tokens regularly

---

## 17. Common Errors (Exam Important)

- Docker permission denied → add jenkins user to docker group
- Git auth failed → wrong PAT scope
- SonarQube not reachable → wrong URL or port 9000 blocked
- Plugin missing → install required plugins
- Pipeline fails → check Jenkins console logs

---

## 18. Key Idea Summary

- Jenkins = automation engine
- Pipeline = CI/CD logic as code
- Credentials = secure secrets storage
- Docker = runtime environment for builds
- SonarQube = code quality gate
- Git + Jenkinsfile = source-driven automation
-
