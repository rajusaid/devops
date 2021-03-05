@Library('nbs-aws-auth-jenkins-sharedlibraries@master') _

pipeline {
     agent { label "jenkins-node" }

     options {
          ansiColor('xterm')
     }

     environment{
          SIGNED_SSH_KEYS_PATH = "$HOME/signed-keys/ocp/$CLUSTER"
          ARTIFACTORY_CREDENTIALS = credentials('ARTIFACTORY_CREDENTIALS')
          ARTIFACTORY_USERNAME = "${env.ARTIFACTORY_CREDENTIALS_USR}"
          ARTIFACTORY_PASSWORD = "${env.ARTIFACTORY_CREDENTIALS_PSW}"
          PROJECT_NAME = "nbs-obc"
     }

     parameters {
          string (
               name: 'CLUSTER',
               defaultValue: 'nbs-obc-test',
               description: 'Environment in which to provision the cluster in')
          string (
              name: 'AWS_ACCOUNT_NAME',
              defaultValue: 'nbs-obc-dev',
              description: 'The name of the aws account which is same as vault namespace')
          booleanParam (
               name: 'DESTROY_ANY_EXISTING_CLUSTER',
               defaultValue: false,
               description: 'Destroy an existing cluster instead of updating it')
          booleanParam (
               name: 'ACTIVE_ACTIVE',
               defaultValue: false,
               description: 'Whether the cluster is being deployed as part of a larger A-A cluster. Used for S3 bucket naming')
     }

    stages {
         
     stage('Preparation') {
          steps {
               script {
                    OCP_DNS_NAME = "${CLUSTER}-openshift.nbs-obc-dev.aws.nbscloud.co.uk"
                    SUB_DOMAIN_OCP_DNS_NAME = "*.${CLUSTER}-openshift-app.nbs-obc-dev.aws.nbscloud.co.uk"
                    cleanWs()
                    checkout([$class: 'GitSCM', branches: [[name: scm.branches[0].name]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
              }
            }
         }
/*
     stage ('Vault Namespace Provisioning') {
          steps {
               build job: 'nbs-cop-vault-configuration-via-terraform', parameters: [
                 string(name: 'CHOOSE_NODE', value: "$CHOOSE_NODE"),
                 string(name: 'VAULT_BASE_NAMESPACE', value: "$AWS_ACCOUNT_NAME"),
                 string(name: 'ENVIRONMENT_NAME', value: "$CLUSTER"),
                 string(name: 'TERRAFORM_OPERATION', value: 'deploy'),
                 booleanParam(name: 'SKIP_VALIDATION', value: true)]
          }
     }
     */
     
     stage ('Generate an SSH key pair and get the public key signed by vault') {
          steps{
                script{
                     CLIENT_TOKEN = token_fetch(aws_account_name: "$AWS_ACCOUNT_NAME", environment_name: "$CLUSTER", vault_url: "$VAULT_URL" )
                    sh """
                    ansible-playbook -vvv $WORKSPACE/vault-signed-ssh/generate-signed-keys.yml --extra-vars "keys_directory=$SIGNED_SSH_KEYS_PATH client_token=$CLIENT_TOKEN  aws_account_name=$AWS_ACCOUNT_NAME vault_url=$VAULT_URL"
                   """
          }
     }
     }
     stage ('Create OCP Cluster Certifcates'){
          steps{
               script{
                    sh """
                    set +x
               
                    OCP_DNS_NAME="${CLUSTER}-openshift.nbs-obc-dev.aws.nbscloud.co.uk"
                    SUB_DOMAIN_OCP_DNS_NAME="*.${CLUSTER}-openshift-app.nbs-obc-dev.aws.nbscloud.co.uk"
                    ansible-playbook $WORKSPACE/vault-certificate/certificate_playbooks/create_cert_playbook/create_cert_playbook.yml --extra-vars "active_active=$ACTIVE_ACTIVE global_cluster_dnsname=null platform=ocp type=cluster dns_name=$OCP_DNS_NAME client_token=$CLIENT_TOKEN env_name=$CLUSTER virtual_env=NA aws_account_name=$AWS_ACCOUNT_NAME vault_url=$VAULT_URL"
                    ansible-playbook $WORKSPACE/vault-certificate/certificate_playbooks/create_cert_playbook/create_cert_playbook.yml --extra-vars "active_active=$ACTIVE_ACTIVE global_cluster_dnsname=null platform=ocp type=cluster dns_name=$SUB_DOMAIN_OCP_DNS_NAME client_token=$CLIENT_TOKEN env_name=$CLUSTER virtual_env=router aws_account_name=$AWS_ACCOUNT_NAME vault_url=$VAULT_URL"
                   
                    set -x
                    """
               }
          }
	}

  stage ('Create OCP ETCD encryption secret'){
       steps{
            script{
               sh """
               set +x

               OCP_DNS_NAME="${CLUSTER}-openshift.nbs-obc-dev.aws.nbscloud.co.uk"
               ansible-playbook $WORKSPACE/vault-certificate/certificate_playbooks/create_cluster_secret_playbook/create_cluster_secret_playbook.yml --extra-vars "platform=ocp type=cluster dns_name=$OCP_DNS_NAME VAULT_TOKEN=$CLIENT_TOKEN env_name=$CLUSTER virtual_env=NA aws_account_name=$AWS_ACCOUNT_NAME vault_url=$VAULT_URL"
               
               set -x
               """
            }
       }
}


     stage ('Replace password placeholders') {
          steps {
               sh "export VAULT_ADDR=https://vault.aws.nbscloud.co.uk; ruby $WORKSPACE/automation_scripts/tokeniser_iac.rb -e $WORKSPACE/configuration-properties/$CLUSTER/iac-config.yml -n $CLUSTER -r eu-west-2 -v $AWS_ACCOUNT_NAME -t $CLIENT_TOKEN"
          }
     }
     
     stage ('Create OCP infrastructure') {
          steps {
               sh '''
               cd $WORKSPACE/terraform/
               if [ "${DESTROY_ANY_EXISTING_CLUSTER}" == "true" ]; then
                    echo "======= Destroying the existing Openshift cluster ======="
                    make destroy project=${PROJECT_NAME} service=openshift environment=${CLUSTER}
               fi
               make deploy project=${PROJECT_NAME} service=openshift environment=${CLUSTER}
               '''
          }
     }

     stage ('Install AWS CloudWatch Agent') {
               steps{
                    sh 'chmod +x $WORKSPACE/cloudwatch-agent/installation/install_cloudwatch_agent.sh'
                    sh '$WORKSPACE/cloudwatch-agent/installation/install_cloudwatch_agent.sh ${CLUSTER} ocp ${WORKSPACE}'
               }
     }
/*
     stage ('Create logs bucket') {
               steps{
                    sh 'sudo sh $WORKSPACE/automation_scripts/createS3Bucket.sh AUDIT_LOGS ${S3_BUCKETS_REGION} ${CLUSTER} ${S3_ACCESS_LOGS_BUCKET} ${AWS_ACCOUNT_NAME} ${ACTIVE_ACTIVE}'
                    sh 'sudo sh $WORKSPACE/automation_scripts/createS3Bucket.sh APP_LOGS ${S3_BUCKETS_REGION} ${CLUSTER} ${S3_ACCESS_LOGS_BUCKET} ${AWS_ACCOUNT_NAME} ${ACTIVE_ACTIVE}'
               }
     }
*/
     stage ('Node Preparation') {
          steps{
               sh '''
                    chmod +x $WORKSPACE/openshift-offline-3.11/scripts/node-preparation.sh
                    sh $WORKSPACE/openshift-offline-3.11/scripts/node-preparation.sh $CLUSTER $AWS_ACCOUNT_NAME $ARTIFACTORY_REDHAT_REGISTRY
               '''
          }
     }

     stage ('Pre-requisites') {
          steps{
               sh 'chmod +x  $WORKSPACE/openshift-offline-3.11/scripts/prerequisites.sh'
               sh 'sh $WORKSPACE/openshift-offline-3.11/scripts/prerequisites.sh $CLUSTER'
          }
     }

     stage ('Installation') {
          steps{
               sh 'chmod +x $WORKSPACE/openshift-offline-3.11/scripts/install.sh'
               sh 'sh $WORKSPACE/openshift-offline-3.11/scripts/install.sh $CLUSTER'
          }
     }

     
     stage ('Cluster Configuration') {
          steps{
               sh "sh $WORKSPACE/openshift-offline-3.11/scripts/configure.sh $CLUSTER $AWS_ACCOUNT_NAME $CLIENT_TOKEN"
	            script{
                      TOKEN = readFile('commandResult').trim()
                      sh "rm -f commandResult"
                      env.TOKEN= TOKEN
                      encrypt("${TOKEN}","${CLUSTER}")

               }
          }
     }
   }

      post {
          always{
               script{
                   sh '''
                        sudo rm -rf /tmp/ocp/cluster/$CLUSTER
                        sudo rm -rf $SIGNED_SSH_KEYS_PATH
                    '''
                    cleanWs()
               }
          }
     }

}

 def encrypt(String token,String clusterid){
 def clusterId=clusterid
   com.cloudbees.plugins.credentials.Credentials c = (com.cloudbees.plugins.credentials.Credentials) new com.openshift.jenkins.plugins.OpenShiftTokenCredentials(com.cloudbees.plugins.credentials.CredentialsScope.GLOBAL,clusterId,clusterId, hudson.util.Secret.fromString(token))
   output = com.cloudbees.plugins.credentials.SystemCredentialsProvider.getInstance().getStore().addCredentials(com.cloudbees.plugins.credentials.domains.Domain.global(), c)
     if( "${output}" == "false")
     println("Updating the Value")

  output1 = com.cloudbees.plugins.credentials.SystemCredentialsProvider.getInstance().getStore().removeCredentials(com.cloudbees.plugins.credentials.domains.Domain.global(), c)
  println("Credentials removed " + output1)
  output2 = com.cloudbees.plugins.credentials.SystemCredentialsProvider.getInstance().getStore().addCredentials(com.cloudbees.plugins.credentials.domains.Domain.global(), c)
  println("Credentials added " + output2)
}
