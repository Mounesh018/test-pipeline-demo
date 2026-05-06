
# Jenkins Pipeline Guide - Docker Hub Integration

## Table of Contents
1. [Overview](#overview)
2. [Complete Pipeline Code](#complete-pipeline-code)
3. [Configuration](#configuration)
4. [Stage Breakdown](#stage-breakdown)
5. [Prerequisites](#prerequisites)
6. [Setup Instructions](#setup-instructions)
7. [Troubleshooting](#troubleshooting)

---

## Overview

This Jenkins pipeline automates the CI/CD process:
- ✅ Pulls code from GitHub
- ✅ Builds Docker images
- ✅ Pushes to Docker Hub
- ✅ Deploys to Kubernetes

---

## Complete Pipeline Code

```groovy
pipeline {
    environment {
        REPOSITORY="https://github.com/Mounesh018"
        GIT_CREDENTIALS="ARC_SSH"
        DOCKER_REGISTRY="docker.io"
        DOCKER_NAMESPACE="your-docker-hub-username"  // Replace with your Docker Hub username
        IMAGE_NAME="${DOCKER_NAMESPACE}/${SERVICE}"
    }
 
    parameters {
        choice(name: 'SERVICE', choices: 'test-pipeline-demo', description: 'Select Frontend')
        string(name: 'BRANCH', defaultValue: 'test', description: 'Provide branch name')
        choice(name: 'DEPLOY_TARGET', choices: 'test-pipeline-demo', description: 'Deploy To')
    }
 
    agent {label "${DEPLOY_TARGET}"}
 
    stages {
        stage ("Git Pull") {
            steps {
                script {
                    dir("${SERVICE}") {
                        git credentialsId: "$GIT_CREDENTIALS", 
                            url: "$REPOSITORY/$SERVICE", 
                            branch: "$BRANCH"
                    }
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    dir("${SERVICE}") {
                        sh 'docker build -f Dockerfile -t ${IMAGE_NAME}:latest -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
                    }
                }
            }
        }
        
        stage('Docker Push to Hub') {
            steps {
                script {
                    dir("${SERVICE}") {
                        withCredentials([usernamePassword(credentialsId: 'docker-hub-cred',
                                                        usernameVariable: 'DOCKER_USER',
                                                        passwordVariable: 'DOCKER_PASS')]) {
                            sh '''
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                docker push ${IMAGE_NAME}:latest
                                docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                            '''
                        }
                    }
                }
            }
        }
 
        stage('Docker Clean') {
            steps {
                script {
                    sh '''
                        docker image rm ${IMAGE_NAME}:latest
                        docker image rm ${IMAGE_NAME}:${BUILD_NUMBER}
                    '''
                }
            }
        }
        
        stage('Deploy to K8s') {
            steps {
                sh '''#!/bin/bash
                    cd /root/k8script/services
                    pwd
                    kubectl delete -f ${SERVICE}-deploy.yml
                    sleep 5
                    kubectl create -f ${SERVICE}-deploy.yml
                '''
            }
        }   
    }
    
    post { 
        always { 
            cleanWs()
        }
    }
}
```

---

## Configuration

### Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `REPOSITORY` | `https://github.com/Mounesh018` | GitHub repository base URL |
| `GIT_CREDENTIALS` | `ARC_SSH` | Jenkins credential ID for Git SSH key |
| `DOCKER_REGISTRY` | `docker.io` | Docker Hub registry (default) |
| `DOCKER_NAMESPACE` | `your-docker-hub-username` | Your Docker Hub username |
| `IMAGE_NAME` | `${DOCKER_NAMESPACE}/${SERVICE}` | Full Docker image name |

### Pipeline Parameters

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `SERVICE` | Choice | test-pipeline-demo | Service/application name |
| `BRANCH` | String | test | Git branch to build |
| `DEPLOY_TARGET` | Choice | test-pipeline-demo | Jenkins agent label |

---

## Stage Breakdown

### Stage 1: Git Pull

**Purpose:** Clone source code from GitHub repository

```groovy
stage ("Git Pull") {
    steps {
        script {
            dir("${SERVICE}") {
                git credentialsId: "$GIT_CREDENTIALS", 
                    url: "$REPOSITORY/$SERVICE", 
                    branch: "$BRANCH"
            }
        }
    }
}
```

**What happens:**
- Creates directory with service name
- Clones repository using SSH credentials
- Checks out specified branch

**Example URL:** `https://github.com/Mounesh018/test-pipeline-demo`

---

### Stage 2: Docker Build

**Purpose:** Build Docker image from Dockerfile

```groovy
stage('Docker Build') {
    steps {
        script {
            dir("${SERVICE}") {
                sh 'docker build -f Dockerfile -t ${IMAGE_NAME}:latest -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }
    }
}
```

**What happens:**
- Reads `Dockerfile` from repository
- Creates image with two tags:
  - `latest` tag (always points to newest build)
  - `BUILD_NUMBER` tag (numbered version for tracking)

**Example Tags:**
```
myusername/test-pipeline-demo:latest
myusername/test-pipeline-demo:42
```

---

### Stage 3: Docker Push to Hub

**Purpose:** Authenticate with Docker Hub and push images

```groovy
stage('Docker Push to Hub') {
    steps {
        script {
            dir("${SERVICE}") {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-cred',
                                                usernameVariable: 'DOCKER_USER',
                                                passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
}
```

**What happens:**
1. Retrieves Docker Hub credentials securely
2. Logs in to Docker Hub (no password shown in logs)
3. Pushes both `latest` and `BUILD_NUMBER` tagged images

**Security Note:** Uses `--password-stdin` to prevent password from being visible in logs

---

### Stage 4: Docker Clean

**Purpose:** Remove local Docker images to free disk space

```groovy
stage('Docker Clean') {
    steps {
        script {
            sh '''
                docker image rm ${IMAGE_NAME}:latest
                docker image rm ${IMAGE_NAME}:${BUILD_NUMBER}
            '''
        }
    }
}
```

**What happens:**
- Removes `latest` tag image locally
- Removes `BUILD_NUMBER` tag image locally
- Images remain on Docker Hub (safe to delete locally)

---

### Stage 5: Deploy to K8s

**Purpose:** Update Kubernetes deployment with new image

```groovy
stage('Deploy to K8s') {
    steps {
        sh '''#!/bin/bash
            cd /root/k8script/services
            pwd
            kubectl delete -f ${SERVICE}-deploy.yml
            sleep 5
            kubectl create -f ${SERVICE}-deploy.yml
        '''
    }
}
```

**What happens:**
1. Changes to Kubernetes manifests directory
2. Deletes old deployment
3. Waits 5 seconds for graceful shutdown
4. Creates new deployment with updated image

**File:** `test-pipeline-demo-deploy.yml`

---

## Prerequisites

### Jenkins Setup
- [ ] Jenkins instance with Docker installed
- [ ] Jenkins agent with label matching `DEPLOY_TARGET`
- [ ] Docker CLI available on agent
- [ ] Kubectl installed on agent

### GitHub Setup
- [ ] SSH key generated and added to GitHub account
- [ ] Repository accessible via SSH

### Docker Hub Setup
- [ ] Docker Hub account created
- [ ] Personal Access Token generated (recommended over password)

### Kubernetes Setup
- [ ] Kubeconfig configured on Jenkins agent
- [ ] Deployment files at `/root/k8script/services/{SERVICE}-deploy.yml`
- [ ] Deployment references Docker Hub image

---

## Setup Instructions

### Step 1: Add SSH Credentials (Git)

1. Go to Jenkins → Manage Jenkins → Credentials
2. Click "New credentials"
3. Fill in:
   - **Kind:** SSH Username with private key
   - **ID:** `ARC_SSH`
   - **Username:** `git`
   - **Private Key:** Paste your GitHub SSH private key
4. Click Save

### Step 2: Add Docker Hub Credentials

1. Go to Jenkins → Manage Jenkins → Credentials
2. Click "New credentials"
3. Fill in:
   - **Kind:** Username with password
   - **ID:** `docker-hub-cred`
   - **Username:** Your Docker Hub username
   - **Password:** Your Docker Hub Personal Access Token
4. Click Save

**To create a Personal Access Token:**
1. Go to Docker Hub → Account Settings → Security
2. Click "New Access Token"
3. Give it a descriptive name
4. Copy the token and use as password in Jenkins

### Step 3: Create Jenkins Pipeline Job

1. Click "New Item"
2. Enter job name
3. Select "Pipeline"
4. Click OK
5. In "Pipeline" section, select "Pipeline script"
6. Paste the complete pipeline code above
7. Replace `your-docker-hub-username` with your actual Docker Hub username
8. Click Save

### Step 4: Update Kubernetes Deployment File

Create `/root/k8script/services/test-pipeline-demo-deploy.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-pipeline-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-pipeline-demo
  template:
    metadata:
      labels:
        app: test-pipeline-demo
    spec:
      containers:
      - name: app
        image: your-docker-hub-username/test-pipeline-demo:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

**Important:** Replace `your-docker-hub-username` with your actual Docker Hub username

---

## Troubleshooting

### Issue: "docker login" fails

**Solution:**
- Verify Docker Hub credentials in Jenkins
- Check if using Personal Access Token (not password)
- Ensure credential ID matches `docker-hub-cred`

```groovy
// Test locally first
echo "YOUR_TOKEN" | docker login -u "your-username" --password-stdin
```

### Issue: Git clone fails with SSH key error

**Solution:**
- Verify SSH key is added to GitHub account
- Check Jenkins credential ID is `ARC_SSH`
- Ensure repository URL is correct format

```bash
# Test SSH connection
ssh -T git@github.com
```

### Issue: Kubernetes deployment fails

**Solution:**
- Check kubeconfig is accessible on Jenkins agent
- Verify deployment file exists at correct path
- Check image name matches in YAML

```bash
# Test kubectl access
kubectl get deployments
```

### Issue: "docker image rm" fails during clean

**Solution:**
- Image might still be in use
- Can safely ignore this error
- Add `|| true` to continue on error

```groovy
sh 'docker image rm ${IMAGE_NAME}:latest || true'
```

### Issue: Out of disk space

**Solution:**
- Docker Clean stage not running
- Manually clean old images

```bash
# On Jenkins agent
docker system prune -a --force
```

---

## Variables Reference

| Variable | Example | Where Used |
|----------|---------|-----------|
| `${SERVICE}` | test-pipeline-demo | All stages |
| `${BRANCH}` | test | Git Pull stage |
| `${DEPLOY_TARGET}` | test-pipeline-demo | Agent label |
| `${BUILD_NUMBER}` | 42 | Docker Build & Push |
| `${IMAGE_NAME}` | myuser/test-pipeline-demo | Docker stages |

---