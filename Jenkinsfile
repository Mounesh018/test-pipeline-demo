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
                        git credentialsId: "$GIT_CREDENTIALS", url: "$REPOSITORY/$SERVICE", branch: "$BRANCH"
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