#!/usr/bin/env groovy

pipeline {
    agent any
    environment {
        ECR_REPO_NAME = 'java-app'
        EC2_SERVER = '35.182.154.128'
        EC2_USER = 'ec2-user'

        // will be set to the location of the SSH key file that is temporarily created
        SSH_KEY_FILE = credentials('ssh-creds-boto3-server')

        ECR_REGISTRY = '099597654282.dkr.ecr.ca-central-1.amazonaws.com'
        DOCKER_USER = 'AWS'
        DOCKER_PWD = credentials('ecr-repo-pwd')
        CONTAINER_PORT = '8080'
        HOST_PORT = '8080'

        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
        AWS_DEFAULT_REGION = 'ca-central-1'
    }
    stages {
        stage('select image version') {
            steps {
               script {
                  echo 'fetching available image versions'
                  def result = sh(script: 'python3 get-images.py', returnStdout: true).trim()
                  // split returns an Array, but choices expects either a List or String, so we do "as List"
                  def tags = result.split('\n') as List
                  version_to_deploy = input message: 'Select version to deploy', ok: 'Deploy', parameters: [choice(name: 'Select version', choices: tags)]
                  // put together the full image name
                  env.DOCKER_IMAGE = "${ECR_REGISTRY}/${ECR_REPO_NAME}:${version_to_deploy}"
                  echo env.DOCKER_IMAGE
               }
            }
        }
        stage('deploying image') {
            steps {
                script {
                   echo 'deploying docker image to EC2...'
                   def result = sh(script: 'python3 deploy.py', returnStdout: true).trim()
                   echo result
                }
            }
        }
        stage('validate deployment') {
            steps {
                script {
                   echo 'validating that the application was deployed successfully...'
                   def result = sh(script: 'python3 validate.py', returnStdout: true).trim()
                   echo result
                }
            }
        }
    }
}
