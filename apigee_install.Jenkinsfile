@Library('nbs-aws-auth-jenkins-sharedlibraries@master') _

pipeline {
   agent { label "ansible-server2" }

   options {
     	ansiColor('xterm')
    }

   environment{
      SIGNED_SSH_KEYS_PATH = "$HOME/signed-keys/apigee/$CLUSTER"
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
      string (
        name: 'ARTIFACTORY_DEPENDENCY_REPO_PATH',
        defaultValue: 'obcomply-iaac',
        description: 'The path to where dependencies are stored in Artifactory')
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
          APIGEE_DNS_NAME = "${CLUSTER}.apigee-mgmtapi.nbs-obc-dev.aws.nbscloud.co.uk"
          cleanWs()
          checkout([$class: 'GitSCM', branches: [[name: scm.branches[0].name]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
          CLIENT_TOKEN = token_fetch(aws_account_name: "$AWS_ACCOUNT_NAME",environment_name: "$CLUSTER", vault_url: "$VAULT_URL" )
        }
      }
    }

    stage ('Generate an SSH key pair and get the public key signed by vault') {
      
      steps{
        script{
          // set +x
          // VAULT_JENKINS_TOKEN=$(aws secretsmanager get-secret-value --secret-id vault-cicd-terraform-approle --region eu-west-2 | jq .SecretString -r | jq '.["secret-id-token"]' -r)
          // VAULT_JENKINS_ROLE_ID=$(aws secretsmanager get-secret-value --secret-id vault-cicd-terraform-approle --region eu-west-2 | jq .SecretString -r | jq '.["role-id"]' -r)
          // set -x
          sh """
          ansible-playbook -vvv $WORKSPACE/vault-signed-ssh/generate-signed-keys.yml --extra-vars "keys_directory=$SIGNED_SSH_KEYS_PATH client_token=$CLIENT_TOKEN aws_account_name=$AWS_ACCOUNT_NAME vault_url=$VAULT_URL"
          """
        }
      }
    }

		stage ('Replace password placeholders') {
			steps {
				sh "export VAULT_ADDR=https://vault.aws.nbscloud.co.uk; ruby $WORKSPACE/automation_scripts/tokeniser_iac.rb -e $WORKSPACE/configuration-properties/$CLUSTER/iac-config.yml -n $CLUSTER -r eu-west-2 -v $AWS_ACCOUNT_NAME -t $CLIENT_TOKEN"
			}
		}
   
		stage ('Create Apigee Cluster Certificate'){
			steps{
				script{
          //set +x
            // VAULT_JENKINS_TOKEN=$(aws secretsmanager get-secret-value --secret-id vault-cicd-approle --region eu-west-2 | jq .SecretString -r | jq '.["secret-id-token"]' -r)
            // VAULT_JENKINS_ROLE_ID=$(aws secretsmanager get-secret-value --secret-id vault-cicd-approle --region eu-west-2 | jq .SecretString -r | jq '.["role-id"]' -r)
            // set -x
          sh """            
            ansible-playbook $WORKSPACE/vault-certificate/certificate_playbooks/create_cert_playbook/create_cert_playbook.yml --extra-vars "active_active=$ACTIVE_ACTIVE global_cluster_dnsname=null platform=apigee type=cluster dns_name=${CLUSTER}.apigee-mgmtapi.nbs-obc-dev.aws.nbscloud.co.uk  client_token=$CLIENT_TOKEN env_name=$CLUSTER virtual_env=NA aws_account_name=$AWS_ACCOUNT_NAME vault_url=$VAULT_URL "
          """
				}
			}
		}

		stage ('AWS: Create Apigee infrastructure') {
			steps {
				sh '''
					cd $WORKSPACE/terraform/

					if [ "${DESTROY_ANY_EXISTING_CLUSTER}" == "true" ]; then
							echo "======= Destroying the existing Apigee cluster ======="
              if [ "${CLUSTER}" == "nbs-obc-mctc" ] || [ "${CLUSTER}" == "nbs-obc-pt" ]; 
              then
							  make destroy project=${PROJECT_NAME} service=apigee-12 environment=${CLUSTER}
              else
                make destroy project=${PROJECT_NAME} service=apigee environment=${CLUSTER}
              fi
					fi

          if [ "${CLUSTER}" == "nbs-obc-mctc" ] || [ "${CLUSTER}" == "nbs-obc-pt" ]; 
          then
            make deploy project=${PROJECT_NAME} service=apigee-12 environment=${CLUSTER}
          else
            make deploy project=${PROJECT_NAME} service=apigee environment=${CLUSTER}
          fi
        '''
        }
      }

    stage ('AWS CloudWatch Configuration') {
      steps{
        sh 'chmod +x $WORKSPACE/cloudwatch-agent/installation/install_cloudwatch_agent.sh'
        sh '$WORKSPACE/cloudwatch-agent/installation/install_cloudwatch_agent.sh ${CLUSTER} apigee ${WORKSPACE}'
      }
    }

/*     
      stage ('Create audit logs bucket') {
        steps{
          sh 'sudo sh $WORKSPACE/automation_scripts/createS3Bucket.sh AUDIT_LOGS ${S3_BUCKETS_REGION} ${CLUSTER} ${S3_ACCESS_LOGS_BUCKET} ${AWS_ACCOUNT_NAME} ${ACTIVE_ACTIVE}'
        }
      }

      stage ('Create app logs bucket') {
        steps{
          sh 'sudo sh $WORKSPACE/automation_scripts/createS3Bucket.sh APP_LOGS ${S3_BUCKETS_REGION} ${CLUSTER} ${S3_ACCESS_LOGS_BUCKET} ${AWS_ACCOUNT_NAME} ${ACTIVE_ACTIVE}'
        }
      }
      
*/
      stage('Run python inventory creator and Apigee installation Ansible Script') {
        steps {
          sh 'sh $WORKSPACE/apigee-installation/install/install.sh $CLUSTER offline $CERT_S3_BUCKET $PROJECT_NAME $ARTIFACTORY_URL $ARTIFACTORY_DEPENDENCY_REPO_PATH $AWS_ACCOUNT_NAME $AUDIT_ES'
        }
      }     
   }
  post{
    always{
      script{
        sh '''
         sudo rm -rf /tmp/apigee/cluster/$CLUSTER
         sudo rm -rf $SIGNED_SSH_KEYS_PATH  
        '''
       // cleanWs()
      }
    }
  }
}
