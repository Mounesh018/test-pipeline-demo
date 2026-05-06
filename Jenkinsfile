pipeline {
    environment {
        REPOSITORY="https://github.com/Mounesh018"
        GIT_CREDENTIALS="ARC_SSH"
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
        stage('docker build') {
            steps {
                script {
                    dir("${SERVICE}") {
                //   sh 'docker build -f Dockerfile.dev -t $SERVICE .'
                  sh 'docker build -f Dockerfile -t $SERVICE .'
                }
            }
            }
        }
        stage('docker push') {
          steps {
            script {
              dir("${SERVICE}") {
                 withCredentials([usernamePassword(credentialsId: 'git-on',
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
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
 
        stage ('Docker Clean'){
            steps {
            script{
                sh 'docker image rm registry.github.com/test-pipeline-demo/apps/$SERVICE:latest'
            }
            }
        }
          stage ('Deploy k8'){
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