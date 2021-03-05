@Library('nbs-aws-auth-jenkins-sharedlibraries@master') _

pipeline {
    agent { label "jenkins-node" }
    options {
     ansiColor('xterm')
    }
    parameters {
       string (
            name: 'VERSION',
            defaultValue: 'latest',
            description: 'IMAGE VERSION')
        choice (
            name: 'BASE_IMAGE_TYPE',
            choices: ['obc-rhel7-jdk18-image'],
            description: 'Perform Specific Image Creation')
        string (
               name: 'CLUSTER',
               defaultValue: 'nbs-obc-dev',
               description: 'Environment in which to provision the cluster in')
          string (
              name: 'AWS_ACCOUNT_NAME',
              defaultValue: 'nbs-obc-dev',
              description: 'The name of the aws account which is same as vault namespace')
     }

    environment {
          ARTIFACTORY_CREDS = credentials('ARTIFACTORY_CREDENTIALS')
            }

    stages {
    stage('Preparation') {
          steps {
               script {
                  cleanWs()
                  checkout([$class: 'GitSCM', branches: [[name: 'feature/base-image-issue']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
                  CLIENT_TOKEN = token_fetch(aws_account_name: "$AWS_ACCOUNT_NAME",environment_name: "$CLUSTER", vault_url: "$VAULT_URL" )            
              }
            }
    }
    stage ('Replace password placeholders') {
          steps {
               sh "export VAULT_ADDR=https://vault.aws.nbscloud.co.uk; ruby $WORKSPACE/automation_scripts/tokeniser_iac.rb -e $WORKSPACE/configuration-properties/$CLUSTER/iac-config.yml -n $CLUSTER -r eu-west-2 -v $AWS_ACCOUNT_NAME -t $CLIENT_TOKEN"
          }
     }
    stage('Creating the image') {

        steps {
            script{
                sh "cd $WORKSPACE/base-images/${BASE_IMAGE_TYPE}/;ls -ltrh;sed -i 's/IMAGE_NAME=value/IMAGE_NAME=${BASE_IMAGE_TYPE}/g' Makefile;sed -i 's/IMAGE_TAG_VERSION=value/IMAGE_TAG_VERSION=${VERSION}/g' Makefile;make"   
                           
            }
        }
    }

    stage('Upload to Artifactory') {
        steps {
            script{
                withCredentials([usernamePassword(credentialsId: 'ARTIFACTORY_CI_CREDENTIALS', passwordVariable: 'ARTIFACTORY_PASSWORD', usernameVariable: 'ARTIFACTORY_USERNAME')]) {
                sh "echo $ARTIFACTORY_PASSWORD | sudo docker login -u $ARTIFACTORY_USERNAME --password-stdin obcomply-docker-rel-local.artifactory.aws.nbscloud.co.uk"
                sh "sudo docker tag ${BASE_IMAGE_TYPE}:${VERSION} obcomply-docker-rel-local.artifactory.aws.nbscloud.co.uk/openshift/${BASE_IMAGE_TYPE}:${VERSION}"   
                sh "sudo docker push obcomply-docker-rel-local.artifactory.aws.nbscloud.co.uk/openshift/${BASE_IMAGE_TYPE}:${VERSION}"   
                }                 
            }
        }
    }
    }
}
