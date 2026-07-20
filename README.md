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

## 🛠️ Installation & Host Provisioning

### 1. Controller System Setup (Jenkins Server Engine)
Execute these commands on your core Ubuntu Linux server to install OpenJDK 17 LTS and configure the official stable Jenkins release repository:

```bash
# Step 1: Install Java Runtime Environment (LTS 17 Dependencies)
sudo apt update
sudo apt install -y openjdk-17-jdk openjdk-17-jre

# Step 2: Import Official Jenkins Debian Signature Keys safely
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  [https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key](https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key)

# Step 3: Append the Stable Repository Package Configurations
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  [https://pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable) binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Step 4: Install and Enable Daemon Services
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
