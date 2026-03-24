# 🚀 Secure CI/CD Automation (DevSecOps Pipeline)

## 📌 Project Overview
This project demonstrates a **production-grade DevSecOps CI/CD pipeline** built using Jenkins with integrated security scanning tools to ensure secure and reliable deployments.

---

## 🏗️ Pipeline Architecture

Code → Build → Test → Security Scans → Docker Build → Image Scan → Deploy

---

## ⚙️ Tech Stack

- CI/CD: Jenkins
- Build Tool: Gradle
- Code Quality: SonarQube
- Secret Detection: GitLeaks
- Dependency Scan: OWASP Dependency-Check
- Container Security: Trivy
- Cloud: AWS (ECR)
- Containerization: Docker
- Deployment: Docker Compose

---

## 📁 Project Structure


Jenkinsfile
docs/
scripts/
screenshots/


---

## 🚀 Pipeline Stages

1. Workspace Cleanup
2. Code Checkout (AWS CodeCommit)
3. GitLeaks Secret Scan
4. Build (Gradle)
5. Unit Testing
6. SonarQube Analysis
7. Quality Gate Check
8. OWASP Dependency Scan
9. Docker Build
10. Trivy Image Scan
11. Trivy Filesystem Scan
12. Push to AWS ECR
13. Deployment via Docker Compose

---

## 🔐 Security Integration

- **GitLeaks** → Detects hardcoded secrets
- **SonarQube** → Code quality & vulnerabilities
- **OWASP Dependency Check** → Dependency risks
- **Trivy** → Image & filesystem vulnerabilities

---

## 🔥 Key Features

- Parameterized pipeline execution
- Fail-fast security checks
- Automated vulnerability detection
- Secure Docker image deployment
- AWS ECR integration

---

## ⚠️ Challenges & Fixes

- Fixed GitLeaks JSON parsing issue
- Handled Trivy false positives
- Integrated SonarQube Quality Gate
- Managed OWASP database caching

---

## 📌 Future Improvements

- Add Kubernetes deployment
- Integrate Slack notifications
- Add approval gates before deploy
- Implement GitHub Actions alternative

---

## 👨‍💻 Author

Hariharan B