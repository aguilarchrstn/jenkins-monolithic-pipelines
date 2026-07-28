---

# 📖 Technical Documentation: AI Code Review & QA Integration

**System:** Main Jenkins CI/CD Pipeline (`LitChat Deployment`)

**Server Environment:** `ubuntu@ip-10-0-1-157` (Jenkins Master Container)

**Deployment Target:** `ubuntu@ip-10-0-1-222` (TEST Environment)

**Author:** Engineering / DevSecOps Team

**Status:** Active / Production

---

## 1. Executive Summary

This documentation details the integration of OpenRouter AI into the Jenkins CI/CD pipeline as an **Advisory Guardrail**.

The goal of this system is to perform automated, real-time code reviews on incoming Git changes during the build phase. It analyzes code diffs for security vulnerabilities (OWASP Top 10), runtime logic bugs, edge cases, and performance bottlenecks, generating a structured Markdown report (`ai_review_report.md`) attached to every build.

To protect pipeline availability, the AI review operates on a **non-blocking advisory model**: static security scanners (Trivy, Syft) serve as hard deployment gates, while AI reviews provide deep qualitative feedback without failing builds on API timeouts or false positives.

---

## 2. Infrastructure & Architectural Design

### Component Overview

```
 +-----------------------------------------------------------------------------------+
 |  Jenkins Master Server (ip-10-0-1-157)                                           |
 |                                                                                   |
 |   +---------------------------------------------------------------------------+   |
 |   | Host Path: /home/ubuntu/ai_review.py                                     |   |
 |   +---------------------------------------------------------------------------+   |
 |                                   │ (Read-Only Mount)                             |
 |                                   ▼                                               |
 |   +---------------------------------------------------------------------------+   |
 |   | Docker Container: jenkins-main                                            |   |
 |   |  - Image: jenkins/jenkins:lts-jdk17                                     |   |
 |   |  - Environment: Python 3, Docker CLI, Java 17                             |   |
 |   |  - Workspace execution: Generates code_changes.diff & ai_review_report.md  |   |
 |   +---------------------------------------------------------------------------+   |
 +-----------------------------------------------------------------------------------+
                                     │
                                     │ (OpenRouter API Request over HTTPS)
                                     ▼
                      +-----------------------------+
                      | OpenRouter API v1           |
                      | Model: openai/gpt-4o-mini   |
                      +-----------------------------+

```

### Key Configuration Decisions

* **Container Volume Mounting:** The script resides at `/home/ubuntu/ai_review.py` on the host server (`10.0.1.157`) and is mapped read-only into `jenkins-main` via Docker Compose (`/home/ubuntu:/home/ubuntu:ro`).
* **Runtime Dependency:** `python3` is installed directly inside the `jenkins-main` container to allow local script execution (`sh 'python3 /home/ubuntu/ai_review.py'`).
* **Secrets Management:** The OpenRouter API key is securely retrieved from Jenkins Credentials Manager (`OPENROUTER_API_KEY`) and injected via pipeline environment variables.

---

## 3. Configuration Setup & Deployment Steps

### Step 1: Docker Compose Configuration (`/home/ubuntu/jenkins-stack/docker-compose.yml`)

Ensure `/home/ubuntu` is mounted read-only into the container service:

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins-main
    restart: always
    privileged: true
    user: root
    ports:
      - "8085:8080"
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
      - /home/ubuntu:/home/ubuntu:ro  # Mount host directory for script access
    environment:
      - TZ=Asia/Manila

volumes:
  jenkins_data:

```

### Step 2: Host Python Script Creation (`/home/ubuntu/ai_review.py`)

Create and set executable permissions for the review script:

```bash
chmod 755 /home/ubuntu/ai_review.py

```

**Script Logic:**

```python
import os
import json
import urllib.request
import sys

def run_analysis():
    api_key = os.getenv('OPENROUTER_API_KEY')
    base_url = os.getenv('AI_BASE_URL', 'https://openrouter.ai/api/v1')
    model = os.getenv('AI_MODEL', 'openai/gpt-4o-mini')

    if not api_key:
        print("[Advisory Check] OPENROUTER_API_KEY missing. Skipping AI review.")
        sys.exit(0)

    diff_text = ''
    if os.path.exists('code_changes.diff'):
        with open('code_changes.diff', 'r', encoding='utf-8') as f:
            diff_text = f.read()

    # Skip API call if diff is empty
    if not diff_text.strip():
        print("[Advisory Check] No code changes detected in diff. Skipping API call.")
        with open('ai_review_report.md', 'w', encoding='utf-8') as report:
            report.write('# AI Code Review & QA Summary\n\nNo code changes detected in this build.')
        sys.exit(0)

    # Safe truncation handling for token limits
    truncated = False
    if len(diff_text) > 8000:
        diff_text = diff_text[:8000]
        truncated = True

    prompt_content = (
        "You are an expert Senior DevSecOps Engineer and Lead QA Analyst.\n"
        "Review the following Git diff for a web application deployment.\n\n"
        "### INSTRUCTIONS:\n"
        "1. Prioritize critical bugs, security vulnerabilities (OWASP Top 10), performance bottlenecks, and breaking changes.\n"
        "2. Do NOT comment on general code formatting, minor style preferences, or missing comments unless they present a functional risk.\n"
        "3. Keep your output concise, highly structured, and directly actionable.\n\n"
        "### REQUIRED OUTPUT FORMAT:\n"
        "## 🚦 Risk Level: [LOW / MEDIUM / HIGH / CRITICAL]\n\n"
        "### ⚠️ Critical Findings & Security Risks\n"
        "- Bullet points of severe bugs, memory leaks, security holes (e.g., unsanitized inputs, exposed secrets, dangerous SQL/shell executions).\n\n"
        "### 🐛 Functional & Logic Flaws\n"
        "- Edge cases, improper error handling, or unhandled promise/async failures.\n\n"
        "### 💡 Architectural & Performance Suggestions\n"
        "- High-impact optimizations (e.g., inefficient database queries, unindexed lookups, container/environment config issues).\n\n"
        "### ✅ Overall QA Summary\n"
        "A 2-sentence summary on whether this change is safe to proceed to testing or requires immediate developer revision.\n\n"
        f"### GIT DIFF TO ANALYZE:\n{diff_text}"
    )

    if truncated:
        prompt_content += "\n\n[NOTE: Diff was truncated at 8000 characters due to size limits.]"

    payload = {
        'model': model,
        'messages': [
            {'role': 'system', 'content': 'You perform concise, critical code reviews focusing on QA, bugs, and security risk.'},
            {'role': 'user', 'content': prompt_content}
        ]
    }

    req = urllib.request.Request(
        f"{base_url}/chat/completions",
        data=json.dumps(payload).encode('utf-8'),
        headers={
            'Authorization': f"Bearer {api_key}",
            'Content-Type': 'application/json',
            'HTTP-Referer': 'https://jenkins.local',
            'X-Title': 'LitChat-Jenkins-Pipeline'
        }
    )

    try:
        with urllib.request.urlopen(req, timeout=30) as response:
            result = json.loads(response.read().decode('utf-8'))
            review = result['choices'][0]['message']['content']
            
            print('\n==================================================')
            print('          OPENROUTER AI CODE ANALYSIS             ')
            print('==================================================\n')
            print(review)
            print('\n==================================================\n')
            
            with open('ai_review_report.md', 'w', encoding='utf-8') as report:
                report.write('# AI Code Review & QA Summary\n\n' + review)
    except Exception as e:
        print(f"[Advisory Warning] OpenRouter API review failed or timed out: {e}")
        with open('ai_review_report.md', 'w', encoding='utf-8') as report:
            report.write(f'# AI Code Review & QA Summary\n\nReview skipped due to API communication issue: {e}')

if __name__ == '__main__':
    run_analysis()

```

### Step 3: Container Environment Preparation

Inside `ubuntu@ip-10-0-1-157`:

```bash
# Restart container to pick up new volume mapping
cd ~/jenkins-stack
docker compose down && docker compose up -d

# Install Python 3 inside container
docker exec -it jenkins-main apt-get update
docker exec -it jenkins-main apt-get install -y python3

```

---

## 4. Pipeline Integration (`Jenkinsfile`)

The pipeline stage utilizes `catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE')` to implement non-blocking execution.

```groovy
stage('🤖 AI Code Review & QA') {
    when { expression { params.PERFORM_ROLLBACK == false } }
    steps {
        script {
            echo "Analyzing code changes with OpenRouter AI..."
            
            catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                // Generate workspace git diff
                sh '''
                    git diff HEAD~1 HEAD > code_changes.diff || git diff HEAD > code_changes.diff || true
                '''

                // Execute host-mounted script
                sh 'python3 /home/ubuntu/ai_review.py'
            }
        }
    }
}

```

**Artifact Archival (`post` block):**

```groovy
post {
    always {
        archiveArtifacts artifacts: 'ai_review_report.md', allowEmptyArchive: true
        echo "Pipeline Run Complete. Sanitizing workspace..."
        cleanWs()
    }
}

```

---

## 5. Maintenance & Operations

### Model Adjustments

To change the underlying LLM, update the `AI_MODEL` environment variable in the `Jenkinsfile` environment block or in `/home/ubuntu/ai_review.py`:

* **Fast / Low Cost (Current Default):** `openai/gpt-4o-mini`
* **Deep Reasoning / Security Intensive:** `anthropic/claude-3.5-sonnet` or `deepseek/deepseek-coder`

### Troubleshooting

| Issue | Root Cause | Solution |
| --- | --- | --- |
| **`python3: not found`** | Container missing Python environment. | Run `docker exec -it jenkins-main apt-get install -y python3`. |
| **`FileNotFoundError: /home/ubuntu/ai_review.py`** | Volume mount missing in Docker Compose. | Verify `- /home/ubuntu:/home/ubuntu:ro` under `volumes` in `docker-compose.yml` and restart the stack. |
| **Stage marked `UNSTABLE**` | API timeout or invalid `OPENROUTER_API_KEY`. | Inspect build console logs for `[Advisory Warning]`. Verify credentials in Jenkins UI under Credentials > System. |
