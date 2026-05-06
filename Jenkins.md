# Jenkins CI/CD Pipeline for React JS Application

## Overview
This Jenkins pipeline automates:
- Source code checkout from GitHub
- Docker image build for React JS application
- Docker image push to registry
- Kubernetes deployment
- Docker image cleanup
- Jenkins workspace cleanup

---

# Jenkins Pipeline Script

```groovy
pipeline {

    environment {
        REPOSITORY="https://github.com/Mounesh018"
        GIT_CREDENTIALS="ARC_SSH"
    }

    parameters {

        choice(
            name: 'SERVICE',
            choices: 'test-pipeline-demo',
            description: 'Select Frontend'
        )

        string(
            name: 'BRANCH',
            defaultValue: 'test',
            description: 'Provide branch name'
        )

        choice(
            name: 'DEPLOY_TARGET',
            choices: 'test-pipeline-demo',
            description: 'Deploy To'
        )
    }

    agent {
        label "${DEPLOY_TARGET}"
    }

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

        stage('docker build') {

            steps {

                script {

                    dir("${SERVICE}") {

                        sh 'docker build -f Dockerfile -t $SERVICE .'
                    }
                }
            }
        }

        stage('docker push') {

            steps {

                script {

                    dir("${SERVICE}") {

                        withCredentials([
                            usernamePassword(
                                credentialsId: 'git-cred',
                                usernameVariable: 'DOCKER_USER',
                                passwordVariable: 'DOCKER_PASS'
                            )
                        ]) {

                            sh '''

                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin registry.gitlab.com

                                docker tag $SERVICE:latest registry.github.com/test-pipeline-demo/apps/$SERVICE

                                docker push registry.github.com/test-pipeline-demo/apps/$SERVICE

                            '''
                        }
                    }
                }
            }
        }

        stage ('Docker Clean') {

            steps {

                script {

                    sh 'docker image rm registry.github.com/test-pipeline-demo/apps/$SERVICE:latest'
                }
            }
        }

        stage ('Deploy k8') {

            steps {

                sh '''#!/bin/bash

                    cd /root/k8script/services

                    pwd

                    kubectl delete -f $SERVICE-deploy.yml

                    sleep 5

                    kubectl create -f $SERVICE-deploy.yml

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

# CI/CD Pipeline Flow

```text
GitHub Repository
        ↓
Jenkins Pipeline
        ↓
Git Pull
        ↓
Docker Build
        ↓
Docker Push
        ↓
Kubernetes Deployment
        ↓
Application Running
```

---

# Step-by-Step Pipeline Explanation

---

# 1. Environment Variables

```groovy
environment {
    REPOSITORY="https://github.com/Mounesh018"
    GIT_CREDENTIALS="ARC_SSH"
}
```

## Purpose
Stores common variables used throughout pipeline.

### Variables

| Variable | Purpose |
|---|---|
| REPOSITORY | GitHub repository URL |
| GIT_CREDENTIALS | Jenkins Git credentials ID |

---

# 2. Pipeline Parameters

```groovy
parameters {
    choice(name: 'SERVICE', choices: 'test-pipeline-demo', description: 'Select Frontend')

    string(name: 'BRANCH', defaultValue: 'test', description: 'Provide branch name')

    choice(name: 'DEPLOY_TARGET', choices: 'test-pipeline-demo', description: 'Deploy To')
}
```

## Purpose
Allows dynamic input while triggering pipeline.

### Parameters

| Parameter | Purpose |
|---|---|
| SERVICE | Application name |
| BRANCH | Git branch name |
| DEPLOY_TARGET | Jenkins agent/server |

---

# 3. Jenkins Agent

```groovy
agent {
    label "${DEPLOY_TARGET}"
}
```

## Purpose
Selects Jenkins node dynamically based on parameter.

### Example
```text
test-pipeline-demo
```

Pipeline executes on matching Jenkins agent.

---

# 4. Git Pull Stage

```groovy
stage ("Git Pull")
```

## Purpose
Pulls source code from GitHub repository.

### Commands Used

```groovy
git credentialsId: "$GIT_CREDENTIALS",
    url: "$REPOSITORY/$SERVICE",
    branch: "$BRANCH"
```

### What Happens?
- Connects to GitHub
- Authenticates using Jenkins credentials
- Clones selected branch

---

# 5. Docker Build Stage

```groovy
stage('docker build')
```

## Purpose
Builds Docker image for React JS application.

### Command

```bash
docker build -f Dockerfile -t $SERVICE .
```

### Example

```bash
docker build -f Dockerfile -t test-pipeline-demo .
```

### What Happens?
- Reads Dockerfile
- Installs dependencies
- Builds React application
- Creates Docker image

---

# 6. Docker Push Stage

```groovy
stage('docker push')
```

## Purpose
Pushes Docker image to registry.

---

## Docker Login

```bash
docker login
```

Uses Jenkins credentials:
- Username
- Password

---

## Docker Tag

```bash
docker tag $SERVICE:latest registry.github.com/test-pipeline-demo/apps/$SERVICE
```

### Purpose
Creates registry-compatible image tag.

---

## Docker Push

```bash
docker push registry.github.com/test-pipeline-demo/apps/$SERVICE
```

### Purpose
Uploads image into Docker registry.

---

# 7. Docker Clean Stage

```groovy
stage ('Docker Clean')
```

## Purpose
Removes local Docker image after push.

### Command

```bash
docker image rm registry.github.com/test-pipeline-demo/apps/$SERVICE:latest
```

### Benefits
- Frees server storage
- Reduces unused images

---

# 8. Kubernetes Deployment Stage

```groovy
stage ('Deploy k8')
```

## Purpose
Deploys React application into Kubernetes cluster.

---

## Commands Used

```bash
cd /root/k8script/services

kubectl delete -f $SERVICE-deploy.yml

sleep 5

kubectl create -f $SERVICE-deploy.yml
```

---

## Deployment Flow

```text
Delete Existing Deployment
            ↓
Wait 5 Seconds
            ↓
Create New Deployment
```

---

# 9. Post Cleanup Stage

```groovy
post {
    always {
        cleanWs()
    }
}
```

## Purpose
Cleans Jenkins workspace after pipeline execution.

### Benefits
- Removes temporary files
- Frees disk space
- Keeps Jenkins clean

---

# Required Jenkins Credentials

| Credential ID | Purpose |
|---|---|
| ARC_SSH | GitHub repository access |
| git-cred | Docker registry login |

---

# Required Software

| Tool | Purpose |
|---|---|
| Jenkins | CI/CD automation |
| Docker | Containerization |
| Kubernetes | Container orchestration |
| kubectl | Kubernetes command tool |
| Git | Source code management |

---

# Required Files

```text
project/
│
├── Dockerfile
├── Jenkinsfile
├── package.json
├── src/
├── public/
└── k8/
```

---

# Kubernetes Deployment YAML Example

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: react-app

spec:
  replicas: 1

  selector:
    matchLabels:
      app: react-app

  template:
    metadata:
      labels:
        app: react-app

    spec:
      containers:
      - name: react-app
        image: registry.github.com/test-pipeline-demo/apps/react-app
        ports:
        - containerPort: 80
```

---

# Complete CI/CD Architecture

```text
Developer Push Code
            ↓
GitHub Repository
            ↓
Jenkins Trigger
            ↓
Build Docker Image
            ↓
Push Docker Image
            ↓
Deploy to Kubernetes
            ↓
React Application Running
```

---

# Advantages of This Pipeline

| Feature | Benefit |
|---|---|
| Automation | Reduces manual deployment |
| Faster Delivery | Quick deployment |
| Consistency | Same deployment every time |
| Scalability | Kubernetes support |
| Clean Environment | Workspace cleanup |

---

# Important Notes

- Docker service must be running
- Kubernetes cluster must be accessible
- Jenkins agent label must exist
- Deployment YAML must exist:
  
```bash
/root/k8script/services
```

- Ensure Docker registry access is available

---

# Summary

This Jenkins CI/CD pipeline:
- Pulls React JS source code
- Builds Docker image
- Pushes image into registry
- Deploys application into Kubernetes
- Cleans Docker images
- Cleans Jenkins workspace
- Provides automated production deployment