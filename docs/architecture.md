# 🏗️ Pipeline Architecture

This pipeline follows a DevSecOps approach:

Developer → Git Commit → Jenkins Pipeline → Security Checks → Deployment

## Flow

1. Code is fetched from AWS CodeCommit
2. Build is triggered using Gradle
3. Multiple security scans are executed:
   - GitLeaks (Secrets)
   - SonarQube (Code Quality)
   - OWASP (Dependencies)
   - Trivy (Containers)
4. Docker image is built and pushed to AWS ECR
5. Application is deployed using Docker Compose