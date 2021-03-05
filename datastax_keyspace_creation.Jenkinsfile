@Library('nbs-aws-auth-jenkins-sharedlibraries@master') _

pipeline {
   agent { label "jenkins-node" }

   options {
     	ansiColor('xterm')
    }

   environment{
      SIGNED_SSH_KEYS_PATH = "$HOME/signed-keys/datastax/$CLUSTER"
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
    }

    stages {
      
    stage('Preparation') {
      steps {
        script {
          cleanWs()
          checkout([$class: 'GitSCM', branches: [[name: scm.branches[0].name]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
          CLIENT_TOKEN = token_fetch(aws_account_name: "$AWS_ACCOUNT_NAME", environment_name: "$CLUSTER", vault_url: "$VAULT_URL" )
        }
      }
    }

    stage ('Generate an SSH key pair and get the public key signed by vault') {      
      steps{
        script{
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

      stage('Run python inventory creator and Datastax installation Ansible Script') {
        steps {
          script {
              sh """
              sh $WORKSPACE/datastax-installation/install/keyspace_creation.sh $CLUSTER $CLIENT_TOKEN
              """
          }
        }
      }
   }

  post{
    always{
      script{
        sh '''
          sudo rm -rf /tmp/datastax/cluster/$CLUSTER
          sudo rm -rf $SIGNED_SSH_KEYS_PATH  
        '''
      }
    }
  }
}