pipeline {
    agent { label "jenkins-node" }
    environment {
            GITHUB_CREDS = credentials('jenkins-github-enterprise')
            COP_APIGEE_USERNAME = credentials('COP_APIGEE_USERNAME')
            COP_APIGEE_PASSWORD = credentials('COP_APIGEE_PASSWORD')
            VAULT_JENKINS_ROLE_ID = credentials('VAULT_JENKINS_ROLE_ID')
            VAULT_JENKINS_TOKEN = credentials('VAULT_JENKINS_TOKEN')
            }
    options {
     ansiColor('xterm')
    }
    parameters {
       string (
            name: 'ENV_NAME',
            defaultValue: 'nbs-caas-offline',
            description: 'Project/Org/Env for OCP/Apigee')
       string (
            name: 'CLUSTER',
            defaultValue: 'nbs-caas-dev' ,
            description: 'OCP/Apigee Cluster')

    }

    stages {
      stage('Preparation') {
        steps {
          script {
            APIGEE_DNS_NAME = "${env.ENV_NAME}.${env.CLUSTER}.apigee-gtw.${DNS}"
            OCP_DNS_NAME = "${env.ENV_NAME}.${env.CLUSTER}-openshift-app.${DNS}"
            cleanWs()
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-enterprise', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
            //checkout([$class: 'GitSCM', branches: [[name: '*/pipeline']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-enterprise', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
            }
          }
        }


        stage('Apigee: Delete Virtual Host') {

            steps {
                    sh 'sh $WORKSPACE/apigee-installation/vhost-destroy/vhost-destroy.sh $ENV_NAME $CLUSTER $COP_APIGEE_USERNAME $COP_APIGEE_PASSWORD $DNS'
                }
            }

        stage('Apigee: Delete TrustStore') {

            steps {
                    sh 'sh $WORKSPACE/apigee-installation/truststore-creation/delete-truststore.sh $ENV_NAME $CLUSTER $COP_APIGEE_USERNAME $COP_APIGEE_PASSWORD $DNS'
                }
            }

       stage('Apigee: Delete KeyStore') {

            steps {
                      sh 'sh $WORKSPACE/apigee-installation/keystore-creation/delete-keystore.sh $ENV_NAME $CLUSTER $COP_APIGEE_USERNAME $COP_APIGEE_PASSWORD $DNS'
                }
            }

      stage('Apigee: Delete MP association with env') {

             steps {
                     sh 'sh $WORKSPACE/apigee-installation/mp/remove-from-mp.sh $ENV_NAME $CLUSTER $COP_APIGEE_USERNAME $COP_APIGEE_PASSWORD $DNS'
               }
           }

     stage('Apigee: Delete env') {

            steps {
                    sh 'sh $WORKSPACE/apigee-installation/env-destroy/env-destroy.sh $ENV_NAME $CLUSTER $COP_APIGEE_USERNAME $COP_APIGEE_PASSWORD $DNS'
              }
         }

    stage('Apigee: Delete org') {

          steps {
                  sh 'sh $WORKSPACE/apigee-installation/org-destroy/org-destroy.sh $ENV_NAME $CLUSTER $COP_APIGEE_USERNAME $COP_APIGEE_PASSWORD $DNS'
            }
         }

    stage ('Replace password placeholders') {

       			steps {
     				sh "ruby $WORKSPACE/automation_scripts/tokeniser_iac.rb -e $WORKSPACE/configuration-properties/$CLUSTER/iac-config.yml -n dev -r eu-west-2"
       		}
       	}


    stage('Apigee: Delete environment from retention policy') {

         steps {
                sh 'sh $WORKSPACE/apigee-installation/postgress_purge/postgress_purge_remove.sh $CLUSTER $ENV_NAME'
             }
          }

      }
    }
