
---


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

```

### 2. Post-Installation Dashboard Unlock

1. Access the web control interface using your browser at `http://YOUR_SERVER_IP:8080`.
2. Extract your temporary administrator lock password via your controller terminal:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

```


3. Select **"Install Suggested Plugins"** to seed primary dependencies.

### 3. Granting Pipeline Permissions

To prevent permission blocks when building Docker image configuration layers within the automation workflow, ensure the `jenkins` system account belongs to the host's native docker socket execution group:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

```

---

## 🔑 Automated SSH Authentication Setup

To run automated remote execution stages smoothly without requiring interactive passwords, create a standard key handshake:

1. **Generate Key Pair:** Execute this command in your terminal tool (either on the Jenkins host or your workstation):
```bash
ssh-keygen -t rsa -b 4096 -f ./jenkins_deployer_key -N ""

```


This creates `jenkins_deployer_key` (**Private Key**) and `jenkins_deployer_key.pub` (**Public Key**).
2. **Jenkins Credentials Ingestion:** Navigate to **Manage Jenkins** ➔ **Credentials** ➔ **Global** ➔ **Add Credentials**:
* **Type:** SSH Username with private key
* **ID:** `ec2-deployer-key`
* **Username:** `ubuntu` *(Maps directly to native EC2 environment administrative profiles)*
* **Private Key:** Select "Enter directly" and paste the *entire* text block from the generated `jenkins_deployer_key` private file.


3. **Target Node Configuration:** Copy your public key string (`jenkins_deployer_key.pub`) and append it directly to a clean line inside the target user configuration file on **both** deployment machines (`10.0.1.222` and `10.0.1.39`):
```bash
nano ~/.ssh/authorized_keys

```



---

## 📋 Production-Ready Pipeline Definitions

### 🔹 Test Pipeline (`Jenkinsfile-test`)

```groovy
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'PERFORM_ROLLBACK', 
            defaultValue: false, 
            description: 'Check this box to trigger an immediate rollback from the latest backup.'
        )
    }

    environment {
        GITHUB_REPO     = "github.com/aguilarchrstn/monolithic-litchat.git"
        REGISTRY_TAG    = "myregistry.local/your-app:${BUILD_NUMBER}"
        TEST_SERVER_IP  = "10.0.1.222"
        SSH_CRED_ID     = "ec2-deployer-key"
    }

    stages {
        stage('🔄 Manual Rollback Execution') {
            when { expression { params.PERFORM_ROLLBACK == true } }
            steps {
                sshagent([SSH_CRED_ID]) {
                    echo "Rollback flag detected! Restoring target environment..."
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER_IP} '
                            LATEST_BACKUP=$(ls -t /home/ubuntu/backups/litchat-backup-*.tar.gz 2>/dev/null | head -n 1)
                            if [ -n "$LATEST_BACKUP" ]; then
                                echo "Restoring from: $LATEST_BACKUP"
                                cd /home/ubuntu/test/monolithic-litchat/litchat 2>/dev/null && docker compose -p litchat-test down || true
                                tar -xzf "$LATEST_BACKUP" -C /home/ubuntu/test/
                                cd /home/ubuntu/test/monolithic-litchat/litchat && docker compose -p litchat-test up -d --build
                                echo "Rollback completed successfully!"
                            else
                                echo "Error: No backup archive found in /home/ubuntu/backups/"
                                exit 1
                            fi
                        '
                    '''
                }
            }
        }

        stage('🚀 Code Checkout') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                git branch: 'main', credentialsId: 'github-token-id', url: "https://${GITHUB_REPO}"
            }
        }

        stage('📋 Source Quality & SBOM') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                echo "Downloading Syft and generating Software Bill of Materials (SBOM)..."
                sh """
                    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b ./bin
                    ./bin/syft dir:. --output spdx-json=sbom.json
                """
                archiveArtifacts artifacts: 'sbom.json', fingerprint: true
            }
        }

        stage('🛡️ SAST Scan') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                echo "Downloading valid Trivy release and running SAST..."
                sh """
                    mkdir -p ./bin
                    if [ ! -f ./bin/trivy ]; then
                        curl -Lo trivy.tar.gz https://github.com/aquasecurity/trivy/releases/download/v0.72.0/trivy_0.72.0_Linux-64bit.tar.gz
                        tar -xzf trivy.tar.gz -C ./bin trivy
                        rm trivy.tar.gz
                        chmod +x ./bin/trivy
                    fi
                    ./bin/trivy fs --scanners vuln,secret,config .
                """
            }
        }

        stage('🔍 Dockerfile Scan') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                echo "Auditing Dockerfile via Trivy Binary..."
                sh "./bin/trivy config litchat/Dockerfile"
            }
        }

        stage('📦 Build Image') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                echo "Compiling image configuration layers..."
                sh "docker build -t ${REGISTRY_TAG} ./litchat"
            }
        }

        stage('🔒 Container Vulnerability Scan') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                echo "Auditing built image via Trivy..."
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${REGISTRY_TAG}"
            }
        }

        stage('💾 Backup TEST Environment') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                sshagent([SSH_CRED_ID]) {
                    echo "Creating timestamped backup of target environment before deployment..."
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER_IP} '
                            mkdir -p /home/ubuntu/backups && \
                            BACKUP_NAME="litchat-backup-$(date +%Y%m%d_%H%M%S).tar.gz" && \
                            if [ -d "/home/ubuntu/test/monolithic-litchat" ]; then
                                tar -czf /home/ubuntu/backups/${BACKUP_NAME} -C /home/ubuntu/test monolithic-litchat && \
                                echo "Backup created successfully: /home/ubuntu/backups/${BACKUP_NAME}"
                            else
                                echo "No existing deployment directory found to back up. Skipping."
                            fi
                        '
                    '''
                }
            }
        }

        stage('🌐 Deploy to TEST') {
            when { expression { params.PERFORM_ROLLBACK == false } }
            steps {
                sshagent([SSH_CRED_ID]) {
                    echo "Deploying updates to LitChat Test Environment..."
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER_IP} "
                            cd /home/ubuntu/test/monolithic-litchat/litchat && \\
                            git pull origin main || true && \\
                            docker compose -p litchat-test up -d --build
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.PERFORM_ROLLBACK == true) {
                    currentBuild.displayName = "#${BUILD_NUMBER} - Rollback"
                } else {
                    currentBuild.displayName = "#${BUILD_NUMBER} - Deployed complete"
                }
            }
        }
        aborted {
            script {
                currentBuild.displayName = "#${BUILD_NUMBER} - Cancelled"
            }
        }
        failure {
            script {
                currentBuild.displayName = "#${BUILD_NUMBER} - Failed"
            }
        }
        always {
            echo "Pipeline Run Complete. Sanitizing workspace..."
            cleanWs()
        }
    }
}

```

### 🔸 Staging Pipeline (`Jenkinsfile-staging`)

```groovy
pipeline {

    agent any

    

    environment {

        // Source Parameter Layout

        GITHUB_REPO     = "github.com/aguilarchrstn/monolithic-litchat.git"

        REGISTRY_TAG    = "myregistry.local/your-app:${BUILD_NUMBER}"

        

        // Target Test Server Network Matrix

        TEST_SERVER_IP  = "10.0.1.222"  // Input your Test AWS VPC Private IP

        SSH_CRED_ID     = "ec2-deployer-key"

    }

    

    stages {

        stage('🚀 Code Checkout') {

            steps {

                // Ensure 'github-token-id' matches the credentials ID you saved in Jenkins for GitHub

                git branch: 'staging', credentialsId: 'github-token-id', url: "https://${GITHUB_REPO}"

            }

        }



     stage('📋 Source Quality & SBOM') {

                steps {

                    echo "Downloading Syft and generating Software Bill of Materials (SBOM)..."

                    sh """

                        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b ./bin

                        ./bin/syft dir:. --output spdx-json=sbom.json

                    """

                    archiveArtifacts artifacts: 'sbom.json', fingerprint: true

                }

            }



        stage('🛡️ SAST Scan') {

                    steps {

                        echo "Downloading valid Trivy release and running SAST..."

                        sh """

                            mkdir -p ./bin

                            

                            if [ ! -f ./bin/trivy ]; then

                                echo "Fetching Trivy static binary..."

                                # Updated to a valid, active release version (v0.72.0)

                                curl -Lo trivy.tar.gz https://github.com/aquasecurity/trivy/releases/download/v0.72.0/trivy_0.72.0_Linux-64bit.tar.gz

                                tar -xzf trivy.tar.gz -C ./bin trivy

                                rm trivy.tar.gz

                                chmod +x ./bin/trivy

                            fi

                            

                            ./bin/trivy fs --scanners vuln,secret,config .

                        """

            }

        }



        stage('🔍 Dockerfile Scan') {

                    steps {

                        echo "Auditing Dockerfile via Trivy Binary..."

                        sh """

                            # 1. Ensure we are using the stable binary we already downloaded in the SAST stage

                            # 2. Point Trivy directly to the litchat directory path where the Dockerfile lives

                            ./bin/trivy config litchat/Dockerfile

                        """

            }

        }



    stage('📦 Build Image') {

                steps {

                    echo "Compiling image configuration layers using litchat context..."

                    // Pointing the context directly to the application directory

                    sh "docker build -t ${REGISTRY_TAG} ./litchat"

            }

        }



        stage('🔒 Container Vulnerability Scan') {

            steps {

                echo "Auditing built image via Trivy..."

                // Image scans don't need local file mounts, so running via Docker here works perfectly!

                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${REGISTRY_TAG}"

            }

        }

        stage('🌐 Deploy to Staging') {

                    steps {

                        sshagent([SSH_CRED_ID]) {

                            echo "Deploying updates to LitChat Staging Environment..."

                            sh """

                                # Changed 'jenkins_deployer' to the default native 'ubuntu' user

                                ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER_IP} "

                                    cd /home/ubuntu/staging/monolithic-litchat/litchat && \

                                    git pull origin staging || true && \

                                    docker compose -p litchat-staging up -d --build

                                "

                            """

                }

            }

        }

    }

    

    post {

        always {

            echo "Pipeline Run Complete. Sanitizing system workspace cache structures..."

            cleanWs()

        }

    }

} 



```

### 🔹 Production Pipeline (`Jenkinsfile-production`)

```groovy
pipeline {

    agent any

    

    environment {

        // Source Parameter Layout

        GITHUB_REPO     = "github.com/aguilarchrstn/monolithic-litchat.git"

        REGISTRY_TAG    = "myregistry.local/your-app:${BUILD_NUMBER}"

        

        // Target Test Server Network Matrix

        TEST_SERVER_IP  = "10.0.1.39"  // Input your Test AWS VPC Private IP

        SSH_CRED_ID     = "ec2-deployer-key"

    }

    

    stages {

        stage('🚀 Code Checkout') {

            steps {

                // Ensure 'github-token-id' matches the credentials ID you saved in Jenkins for GitHub

                git branch: 'prod', credentialsId: 'github-token-id', url: "https://${GITHUB_REPO}"

            }

        }



     stage('📋 Source Quality & SBOM') {

                steps {

                    echo "Downloading Syft and generating Software Bill of Materials (SBOM)..."

                    sh """

                        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b ./bin

                        ./bin/syft dir:. --output spdx-json=sbom.json

                    """

                    archiveArtifacts artifacts: 'sbom.json', fingerprint: true

                }

            }



        stage('🛡️ SAST Scan') {

                    steps {

                        echo "Downloading valid Trivy release and running SAST..."

                        sh """

                            mkdir -p ./bin

                            

                            if [ ! -f ./bin/trivy ]; then

                                echo "Fetching Trivy static binary..."

                                # Updated to a valid, active release version (v0.72.0)

                                curl -Lo trivy.tar.gz https://github.com/aquasecurity/trivy/releases/download/v0.72.0/trivy_0.72.0_Linux-64bit.tar.gz

                                tar -xzf trivy.tar.gz -C ./bin trivy

                                rm trivy.tar.gz

                                chmod +x ./bin/trivy

                            fi

                            

                            ./bin/trivy fs --scanners vuln,secret,config .

                        """

            }

        }



        stage('🔍 Dockerfile Scan') {

                    steps {

                        echo "Auditing Dockerfile via Trivy Binary..."

                        sh """

                            # 1. Ensure we are using the stable binary we already downloaded in the SAST stage

                            # 2. Point Trivy directly to the litchat directory path where the Dockerfile lives

                            ./bin/trivy config litchat/Dockerfile

                        """

            }

        }



    stage('📦 Build Image') {

                steps {

                    echo "Compiling image configuration layers using litchat context..."

                    // Pointing the context directly to the application directory

                    sh "docker build -t ${REGISTRY_TAG} ./litchat"

            }

        }



        stage('🔒 Container Vulnerability Scan') {

            steps {

                echo "Auditing built image via Trivy..."

                // Image scans don't need local file mounts, so running via Docker here works perfectly!

                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${REGISTRY_TAG}"

            }

        }

        stage('🌐 Deploy to Prodution') {

                    steps {

                        sshagent([SSH_CRED_ID]) {

                            echo "Deploying updates to LitChat Prodution Environment..."

                            sh """

                                # Changed 'jenkins_deployer' to the default native 'ubuntu' user

                                ssh -o StrictHostKeyChecking=no ubuntu@${TEST_SERVER_IP} "

                                    cd /home/ubuntu/prod/monolithic-litchat/litchat && \

                                    git pull origin prod || true && \

                                    docker compose -p litchat-prod up -d --build

                                "

                            """

                }

            }

        }

    }

    

    post {

        always {

            echo "Pipeline Run Complete. Sanitizing system workspace cache structures..."

            cleanWs()

        }

    }

} 


```

---

## 🔍 Dashboard UI Organization Tips

* **Enforcing Layout Order:** Jenkins automatically sorts list views alphabetically. To force an explicit workflow order (Test ➔ Staging ➔ Production) without altering internal pipeline script code, name your Jenkins items using numerical prefixes:
* `01-Litchat-Test`
* `02-Litchat-Staging`
* `03-Litchat-Production`


* **Grouping Environments:** Install the **Folders Plugin** from your plugin management space. This lets you construct a single root directory folder titled `LitChat` to cleanly contain your individual lifecycles away from other projects.

---

## 🩺 Critical Troubleshooting Matrix

| Error Message / Symptom | Root Cause Analysis | Corrective Action Strategy |
| --- | --- | --- |
| `Permission denied (publickey)` | The remote node validation blocks rejected the credentials data transmitted by Jenkins. | 1. Ensure your SSH block targets the administrative user string `ubuntu` rather than custom local identities.<br>

<br>2. Check for missing lines or corrupted block data inside the target host's `~/.ssh/authorized_keys` file. |
| `No such DSL method 'sshagent'` | The execution system is evaluating a wrapper command context without its supporting plugin driver engine active. | Go to **Manage Jenkins** ➔ **Plugins**, look up the **SSH Agent Plugin** in the available indexing library, trigger installation, and restart the system. |
| Environment Containers Disappearing | Multiple deployment setups share identical base system workspace naming structures, forcing Docker to replace running instances. | Enforce isolated container namespaces by applying explicit project flag identifiers (`-p`) inside deployment wrappers (e.g., `docker compose -p litchat-staging up -d --build`). |
| System Networking Socket Port Clash | Two running application stacks attempt to claim binding rights to the exact same physical host port mapping. | Modify the host-side exposed ports inside your project `docker-compose.yml` assets so separate instances utilize different host channels (e.g., Test listening on `3000`, Staging on `3001`). |

```
***

```
