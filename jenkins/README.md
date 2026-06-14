# Jenkins CI/CD Guide

A practical guide to setting up Jenkins for the Landmark Technologies web application.

---

## Table of Contents

1. [Installing Jenkins on Amazon Linux](#installing-jenkins-on-amazon-linux)
2. [Accessing Jenkins](#accessing-jenkins)
3. [Installing Default Plugins](#installing-default-plugins)
4. [Setting Up Credentials](#setting-up-credentials)
5. [Tools Required on Jenkins Server](#tools-required-on-jenkins-server)
6. [Pipeline Types and Examples](#pipeline-types-and-examples)
7. [Creating a Pipeline for This Application](#creating-a-pipeline-for-this-application)
8. [Advanced Jenkins](#advanced-jenkins)

---

## Installing Jenkins on Amazon Linux

### Prerequisites

- Amazon Linux 2023 or Amazon Linux 2 EC2 instance (t3.medium or larger)
- Security group allowing inbound on ports **22** (SSH) and **8080** (Jenkins)

### Step 1: Install Java

```bash
# Amazon Linux 2023
sudo dnf install java-17-amazon-corretto -y

# Amazon Linux 2
sudo amazon-linux-extras install java-openjdk17 -y

# Verify
java -version
```

### Step 2: Install Jenkins

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Amazon Linux 2023
sudo dnf install jenkins -y

# Amazon Linux 2
sudo yum install jenkins -y
```

### Step 3: Start Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

---

## Accessing Jenkins

### Step 1: Get Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 2: Open Browser

Navigate to:
```
http://<your-ec2-public-ip>:8080
```

### Step 3: Unlock Jenkins

Paste the admin password from Step 1 into the "Unlock Jenkins" screen.

### Step 4: Install Suggested Plugins

Select **Install suggested plugins** and wait for installation to complete.

### Step 5: Create Admin User

Fill in username, password, full name, and email. Click **Save and Continue**.

### Step 6: Set Jenkins URL

Set to `http://<your-ec2-public-ip>:8080/` and click **Save and Finish**.

---

## Installing Default Plugins

Go to **Manage Jenkins → Plugins → Available plugins** and install:

| Plugin | Purpose |
|--------|---------|
| **NodeJS** | Auto-installs Node.js for builds |
| **Docker Pipeline** | `docker.build()`, `docker.withRegistry()` |
| **Docker** | Docker agent support |
| **Kubernetes CLI** | Auto-installs kubectl |
| **Pipeline: AWS Steps** | `withAWS` credential injection |
| **AWS Credentials** | AWS credential type |
| **GitHub Integration** | Webhooks & commit status |
| **Workspace Cleanup** | Clean workspace between builds |

Restart Jenkins after installing.

### Configure NodeJS Tool

1. **Manage Jenkins → Tools → NodeJS installations → Add NodeJS**
2. Name: `NodeJS-18`
3. ✅ Install automatically → version `18.x`
4. **Save**

---

## Setting Up Credentials

Go to **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

### 1. GitHub Token

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Secret | Your GitHub PAT (`repo` scope) |
| ID | `github-token` |

### 2. Docker Hub

| Field | Value |
|-------|-------|
| Kind | Username with password |
| Username | Your Docker Hub username |
| Password | Docker Hub access token |
| ID | `dockerhub-creds` |

### 3. AWS Credentials

| Field | Value |
|-------|-------|
| Kind | AWS Credentials |
| Access Key ID | Your IAM access key |
| Secret Access Key | Your IAM secret key |
| ID | `aws-creds` |

---

## Tools Required on Jenkins Server

```bash
# Git
sudo dnf install git -y

# Docker
sudo dnf install docker -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Provided by plugins (no server install needed):**
- Node.js → **NodeJS** plugin
- kubectl → **Kubernetes CLI** plugin

---

## Pipeline Types and Examples

Jenkins supports two pipeline types. Both are defined in a `Jenkinsfile` stored in your repository.

### Declarative Pipeline (Recommended)

Structured syntax with predefined blocks. Easier to read and maintain.

```groovy
pipeline {
    agent any

    tools {
        nodejs 'NodeJS-18'
    }

    environment {
        APP_NAME = 'landmark-web-app'
    }

    stages {
        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'echo Deploying...'
            }
        }
    }

    post {
        success { echo 'Build passed!' }
        failure { echo 'Build failed!' }
        always  { cleanWs() }
    }
}
```

**Key directives:**

| Directive | Purpose |
|-----------|---------|
| `agent` | Where to run (any, docker, label) |
| `tools` | Auto-install tools (nodejs, maven, jdk) |
| `environment` | Set environment variables |
| `stages` / `stage` | Named phases of the pipeline |
| `steps` | Commands to execute |
| `when` | Conditional execution (branch, expression) |
| `post` | Actions after pipeline (success, failure, always) |
| `parallel` | Run stages concurrently |

### Scripted Pipeline

Full Groovy scripting. More flexible, harder to maintain.

```groovy
node {
    stage('Build') {
        checkout scm
        def nodeHome = tool name: 'NodeJS-18', type: 'nodejs'
        env.PATH = "${nodeHome}/bin:${env.PATH}"
        sh 'npm ci'
        sh 'npm run build'
    }

    stage('Test') {
        sh 'npm test'
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh 'echo Deploying...'
        }
    }
}
```

### Declarative vs Scripted

| Feature | Declarative | Scripted |
|---------|-------------|----------|
| Syntax | Structured `pipeline {}` | Free-form `node {}` |
| Learning curve | Low | High |
| Conditionals | `when {}` directive | `if/else` |
| Error handling | `post { failure {} }` | `try/catch` |
| Restart from stage | ✅ | ❌ |
| Recommended | ✅ Most use cases | Complex dynamic logic only |

### Creating a Pipeline Job

**Option A: Multibranch Pipeline (Recommended)**

1. Jenkins Dashboard → **New Item**
2. Name: `landmark-web-app`
3. Select **Multibranch Pipeline** → OK
4. Branch Sources → Add → **GitHub**
5. Credentials: `github-token`
6. Repository URL: `https://github.com/CHAFAH/landmark-web-app.git`
7. Build Configuration: by Jenkinsfile, path: `Jenkinsfile`
8. Save

Jenkins auto-discovers all branches with a `Jenkinsfile` and builds them.

**Option B: Regular Pipeline**

1. Jenkins Dashboard → **New Item**
2. Name: `landmark-deploy`
3. Select **Pipeline** → OK
4. Pipeline → Definition: **Pipeline script from SCM**
5. SCM: Git → URL: `https://github.com/CHAFAH/landmark-web-app.git`
6. Branch: `*/main`
7. Script Path: `Jenkinsfile`
8. Save

---

## Creating a Pipeline for This Application

When you click **New Item** in Jenkins, you see these job types. Here they are from simplest to most advanced:

---

### 1. Freestyle Project

**Definition:** A simple, UI-configured job. You fill in forms to define source code, build steps, and post-build actions. No coding required.

**When to use:** Simple build/test tasks, running shell scripts, learning Jenkins.

**Webhook support:** ✅ Yes

**Setup for Landmark app:**

1. Jenkins Dashboard → **New Item** → Name: `landmark-freestyle` → **Freestyle project** → OK
2. **Source Code Management:** Git → URL: `https://github.com/CHAFAH/landmark-web-app.git` → Branch: `*/main`
3. **Build Environment:** ✅ Provide Node & npm bin/folder to PATH → NodeJS-18
4. **Build Triggers:** ✅ **GitHub hook trigger for GITScm polling**
5. **Build Steps → Add → Execute shell:**

```bash
echo "=== Installing Dependencies ==="
npm ci

echo "=== Running Tests ==="
npm test

echo "=== Building Frontend ==="
npm run build

echo "=== Building Docker Image ==="
docker build -t chafah/landmark-web-app:latest .

echo "=== Pushing to Docker Hub ==="
echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
docker push chafah/landmark-web-app:latest
```

6. Click **Save**

**Limitations:** No version control of the job config, no conditional stages, no parallel execution.

---

### 2. Pipeline

**Definition:** A job defined by a Groovy script (Jenkinsfile) that describes the entire build/test/deploy process as code. Supports stages, conditions, parallel execution, and post-actions.

**When to use:** Any real CI/CD workflow. This is the standard for production pipelines.

**Webhook support:** ✅ Yes

**Setup for Landmark app:**

1. Jenkins Dashboard → **New Item** → Name: `landmark-pipeline` → **Pipeline** → OK
2. **Pipeline → Definition:** Pipeline script from SCM
3. **SCM:** Git → URL: `https://github.com/CHAFAH/landmark-web-app.git` → Branch: `*/main`
4. **Script Path:** `Jenkinsfile`
5. **Build Triggers:** ✅ **GitHub hook trigger for GITScm polling**
6. Click **Save**

Jenkins reads the `Jenkinsfile` from the repo and executes it:

```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-18'
    }
    environment {
        DOCKER_REPO = 'chafah/landmark-web-app'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install & Test') {
            steps {
                sh 'npm ci'
                sh 'npm test'
                sh 'cd server && npm ci && npm test'
            }
        }
        stage('Build') {
            steps { sh 'npm run build' }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("${DOCKER_REPO}:latest")
                        app.push()
                    }
                }
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
        always  { cleanWs() }
    }
}
```

**Advantages over Freestyle:** Version-controlled, supports conditions (`when`), parallel stages, restart from stage, code review via PRs.

---

### 3. Multibranch Pipeline

**Definition:** Automatically scans a repository for branches that contain a `Jenkinsfile` and creates a pipeline job for each branch.

**When to use:** The recommended approach for real projects with multiple branches.

**Webhook support:** ✅ Yes (auto-configured — webhook triggers a branch scan)

**Setup for Landmark app:**

1. Jenkins Dashboard → **New Item** → Name: `landmark-web-app` → **Multibranch Pipeline** → OK
2. **Branch Sources → Add source → GitHub:**
   - Credentials: `github-token`
   - Repository URL: `https://github.com/CHAFAH/landmark-web-app.git`
3. **Behaviours → Add → Filter by name (with regular expression):**
   - Regular expression: `(main|release.*)`
   - This ensures only `main` and `release*` branches are built
4. **Build Configuration:**
   - Mode: by Jenkinsfile
   - Script Path: `Jenkinsfile`
5. Click **Save**

> **Note:** For Multibranch Pipelines, you don't need to check a "Build Triggers" box. The GitHub webhook automatically triggers a branch scan when a push happens.

**Trigger by push (GitHub Webhook):**

1. GitHub repo → Settings → Webhooks → Add webhook
2. Payload URL: `http://<jenkins-ip>:8080/github-webhook/`
3. Content type: `application/json`
4. Events: "Just the push event"
5. Click **Add webhook**

Now any push to `main` or `release*` triggers the pipeline automatically.

**Jenkinsfile:**

```groovy
pipeline {
    agent any
    tools { nodejs 'NodeJS-18' }
    environment {
        DOCKER_REPO = 'chafah/landmark-web-app'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'landmark-eks'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install & Test') {
            steps {
                sh 'npm ci && npm test'
                sh 'cd server && npm ci && npm test'
            }
        }
        stage('Build') {
            steps { sh 'npm run build' }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    def branch = env.BRANCH_NAME.replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}"
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("${DOCKER_REPO}:${IMAGE_TAG}")
                        app.push()
                    }
                }
            }
        }
        stage('Deploy to Staging') {
            when { branch pattern: 'release*', comparator: 'GLOB' }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: staging/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: staging/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: production/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: production/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
        always  { cleanWs() }
    }
}
```

**What happens:**
- Push to `main` → test → build → docker push → deploy to **production**
- Push to `release*` → test → build → docker push → deploy to **staging**

---

### 4. Folder

**Definition:** An organizational container that groups related jobs together. Folders can have their own credentials, views, and permissions. They don't run builds themselves.

**When to use:** Organizing jobs by team, project, or environment.

**Webhook support:** ❌ No (folders don't run builds)

**Setup:**

1. Jenkins Dashboard → **New Item** → Name: `landmark-technologies` → **Folder** → OK
2. Inside the folder, create your pipeline jobs
3. Optionally scope credentials to this folder only

**Structure example:**
```
landmark-technologies/          (Folder)
├── landmark-web-app            (Multibranch Pipeline)
├── landmark-api-service        (Multibranch Pipeline)
└── landmark-infrastructure     (Pipeline)
```

---

### 5. Multi-configuration Project (Matrix Project)

**Definition:** Runs the same build across multiple configurations (combinations of axes). For example, testing on multiple OS versions, Java versions, or browser types simultaneously.

**When to use:** Cross-platform testing, compatibility matrices.

**Webhook support:** ✅ Yes (same as Freestyle — check "GitHub hook trigger for GITScm polling")

**Setup for Landmark app (test on Node 18 and Node 20):**

1. Jenkins Dashboard → **New Item** → Name: `landmark-matrix` → **Multi-configuration project** → OK
2. **Configuration Matrix → Add axis → User-defined:**
   - Name: `NODE_VERSION`
   - Values: `18 20`
3. **Build Steps → Execute shell:**

```bash
# Install the specific Node version
nvm install $NODE_VERSION
nvm use $NODE_VERSION
node --version

npm ci
npm test
```

Jenkins creates a build for each value: one with Node 18, one with Node 20.

**Note:** Multi-configuration projects are older. For most cases, use a Declarative Pipeline with a `matrix` directive instead:

```groovy
pipeline {
    agent none
    stages {
        stage('Test Matrix') {
            matrix {
                axes {
                    axis {
                        name 'NODE_VERSION'
                        values '18', '20'
                    }
                }
                stages {
                    stage('Test') {
                        agent { docker { image "node:${NODE_VERSION}" } }
                        steps {
                            sh 'npm ci && npm test'
                        }
                    }
                }
            }
        }
    }
}
```

---

### Summary: Which Job Type to Use

| Job Type | Use Case | Webhook Support | Recommended? |
|----------|----------|:---------------:|--------------|
| Freestyle | Simple scripts, learning Jenkins | ✅ | ❌ Not for production |
| Pipeline | Single-branch CI/CD | ✅ | ✅ Good |
| Multibranch Pipeline | Multi-branch CI/CD with auto-discovery | ✅ (auto) | ✅ Best for most projects |
| Folder | Organizing multiple jobs | ❌ | ✅ Always use for structure |
| Multi-configuration | Cross-platform matrix testing | ✅ | ⚠️ Use pipeline `matrix` instead |

For the Landmark project, we use **Multibranch Pipeline** inside a **Folder**.

---

### Setting Up GitHub Webhook (Required for All Webhook-Enabled Jobs)

This is a one-time setup per repository. The same webhook works for all Jenkins jobs that monitor this repo.

**Prerequisites:**
- Jenkins must be accessible from the internet (public IP or domain)
- The **GitHub Integration** plugin must be installed

**Steps:**

1. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
2. Fill in:

| Field | Value |
|-------|-------|
| Payload URL | `http://<jenkins-public-ip>:8080/github-webhook/` |
| Content type | `application/json` |
| Secret | (leave empty or set a secret for verification) |
| Which events? | "Just the push event" |
| Active | ✅ |

3. Click **Add webhook**
4. GitHub sends a test ping — verify it shows a ✅ green check under **Recent Deliveries**

**How it works:**
```
Developer pushes to GitHub
        │
        ▼
GitHub sends POST to http://<jenkins-ip>:8080/github-webhook/
        │
        ▼
Jenkins receives the webhook
        │
        ├── Freestyle/Pipeline: triggers build if "GitHub hook trigger" is checked
        └── Multibranch: triggers branch scan → builds the pushed branch
```

**Troubleshooting webhooks:**

| Problem | Check |
|---------|-------|
| ❌ Red X in Recent Deliveries | Jenkins not reachable — check Security Group allows 8080 from `0.0.0.0/0` |
| Webhook delivered but no build | Freestyle/Pipeline: ensure "GitHub hook trigger for GITScm polling" is checked |
| Multibranch not building | Check branch filter regex matches the pushed branch |
| Response 403 | CSRF issue — ensure GitHub Integration plugin is installed |
| Response timeout | Jenkins is behind a firewall or NAT — needs public access |

---

### Pipeline Flow

```
┌──────────┐    ┌──────────────┐    ┌───────────┐    ┌─────────────┐    ┌──────────────────┐
│ Checkout │───▶│ Install/Test │───▶│ Build App │───▶│ Docker Push │───▶│ Deploy to K8s    │
└──────────┘    └──────────────┘    └───────────┘    └─────────────┘    └──────────────────┘
                                                      (deploy branches)  (branch-specific ns)
```

### Branch Deployment Strategy

| Branch | CI (test + build) | Docker Push | Deploy To |
|--------|:-----------------:|:-----------:|-----------|
| feature/* | ✅ | ❌ | — |
| develop | ✅ | ✅ | dev namespace |
| release/* | ✅ | ✅ | staging namespace |
| main | ✅ | ✅ | production namespace |
| hotfix/* | ✅ | ✅ | production namespace |

---

## Advanced Jenkins

### Shared Libraries

Reuse pipeline code across projects:

```groovy
// In Jenkinsfile
@Library('landmark-shared') _

pipeline {
    stages {
        stage('Deploy') {
            steps { deployToK8s(namespace: 'production', image: env.IMAGE_TAG) }
        }
    }
}
```

### Parallel Stages

```groovy
stage('Tests') {
    parallel {
        stage('Frontend Tests') { steps { sh 'npm test' } }
        stage('Server Tests') { steps { sh 'cd server && npm test' } }
    }
}
```

### Manual Approval

```groovy
stage('Approve Production') {
    when { branch 'main' }
    steps { input message: 'Deploy to production?', ok: 'Deploy' }
}
```

### Notifications (Slack)

```groovy
post {
    success { slackSend color: 'good', message: "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} passed" }
    failure { slackSend color: 'danger', message: "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} failed" }
}
```

### Jenkins Agents

Run builds on separate machines:

1. **Manage Jenkins → Nodes → New Node**
2. Configure SSH connection to agent EC2
3. Set labels (e.g., `docker`, `linux`)
4. Use in pipeline: `agent { label 'docker' }`

### Backup Jenkins

```bash
#!/bin/bash
BACKUP_DIR="/opt/jenkins-backups"
DATE=$(date +%Y%m%d-%H%M%S)

sudo tar -czf ${BACKUP_DIR}/jenkins-backup-${DATE}.tar.gz \
    --exclude="/var/lib/jenkins/workspace" \
    --exclude="/var/lib/jenkins/war" \
    /var/lib/jenkins/

# Keep last 7 days
find ${BACKUP_DIR} -name "*.tar.gz" -mtime +7 -delete
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `docker: permission denied` | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `npm: command not found` | Install NodeJS plugin, configure in Tools → NodeJS-18 |
| `kubectl: command not found` | Install **Kubernetes CLI** plugin — it provides kubectl |
| `aws: command not found` | Install AWS CLI on the server |
| Pipeline not triggering | Check webhook in GitHub Settings → Webhooks → Recent Deliveries |
| `Authentication failed` for Git | Regenerate PAT, update credential in Jenkins |

---

## References

- [Jenkins Official Docs](https://www.jenkins.io/doc/)
- [Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins Plugin Index](https://plugins.jenkins.io/)
