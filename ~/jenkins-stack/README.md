# 🚀 LitChat Multi-Tier CI/CD Pipeline Infrastructure Reference

This repository houses the complete documentation, architecture mapping, and native Jenkinsfile pipeline definitions used to build, security-scan, and deploy **LitChat** across Test, Staging, and Production environments.

---

## 🏗️ Environment Infrastructure Matrix

The pipeline logic orchestrates deployments across isolated application paths and targets specific branches and server nodes:

*   **Test Environment:** Tracks the `main` branch, deploys to server `10.0.1.222`, isolates containers under the `-p litchat-test` flag, and targets host path `/home/ubuntu/test/monolithic-litchat/litchat`.
*   **Staging Environment:** Tracks the `staging` branch, deploys to server `10.0.1.222`, isolates containers under the `-p litchat-staging` flag, and targets host path `/home/ubuntu/staging/monolithic-litchat/litchat`.
*   **Production Environment:** Tracks the `prod` branch, deploys to a dedicated node `10.0.1.39`, isolates containers under the `-p litchat-prod` flag, and targets host path `/home/ubuntu/prod/monolithic-litchat/litchat`.

> ⚠️ **Important Architecture Note:** Test and Staging share the same underlying EC2 server instance (`10.0.1.222`). To run them concurrently without configuration overlapping or container runtime collisions, they enforce strict directory separation and unique Docker Compose project names (`-p`). Production sits completely isolated on a separate node (`10.0.1.39`).

---

## 🛠️ Controller Provisioning via Docker Compose

### 1. Host Preparation
Because Jenkins runs as a containerized stack but needs to execute sibling Docker commands (`docker build`, `docker run`) on the host engine, you must grant the appropriate permissions to the host's Docker socket before spinning up the stack:
```bash
sudo chmod 666 /var/run/docker.sock


2. Stack Deployment (~/jenkins-stack/docker-compose.yml)
The Jenkins controller is deployed on instance 10.0.1-157 inside ~/jenkins-stack using the following production-ready configuration:

YAML
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins-main
    restart: always
    privileged: true
    user: root
    ports:
      - "8085:8080"  # Jenkins Dashboard accessed via port 8085
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    environment:
      - TZ=Asia/Manila

volumes:
  jenkins_data:
To launch or restart this stack, navigate to the directory and run:

Bash
cd ~/jenkins-stack
docker compose up -d
3. Setup & Unlock
Access the web dashboard via your browser at http://10.0.1.157:8085.

Retrieve the initial setup administrator password directly from the container logs:

Bash
docker logs jenkins-main
Select "Install Suggested Plugins" to populate the core ecosystem.
