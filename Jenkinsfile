pipeline {
    environment {
        REPOSITORY="git@gitlab.com:tenjinonline/apps"
        GIT_CREDENTIALS="ARC_SSH"
    }
 
    parameters {
        choice(name: 'SERVICE', choices: 'tplus-frontend\ncos-frontend\ntplus-mfe-nginx\ntenjin-online\nto-customer-onboarding-web\ncms-nginx', description: 'Select Frontend')
        string(name: 'BRANCH', defaultValue: 'release-2.0', description: 'Provide branch name')
        choice(name: 'DEPLOY_TARGET', choices: 'tenjin-online-test-1', description: 'Deploy To')
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
                        docker tag $SERVICE:latest registry.gitlab.com/tenjinonline/apps/$SERVICE
                        docker push registry.gitlab.com/tenjinonline/apps/$SERVICE
                    '''
                }
            }
        }
    }
}
 
        stage ('Docker Clean'){
            steps {
            script{
                sh 'docker image rm registry.gitlab.com/tenjinonline/apps/$SERVICE:latest'
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