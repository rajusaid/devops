@Library('nbs-aws-auth-jenkins-sharedlibraries@master') _

import groovy.json.JsonOutput;

def vault_url = "https://vault.aws.nbscloud.co.uk"
def clusterName
def credentialData
def ocToken = ""
def appDInt = false
def appDVersion = ""
def imageTag = "promote"
def openshiftNamespace = "dummy"

pipeline {
   agent {
       label "jenkins-node"
   }

   environment{
      SIGNED_SSH_KEYS_PATH = "$HOME/signed-keys/ocp/$CLUSTER"
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
      stage("Git Checkout") {
          steps {
              script {
                checkout([$class: 'GitSCM', branches: [[name: scm.branches[0].name]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
              }
          }
      }
      stage('Get Vault Credential') {
         steps {
             script {
                 clusterName = params.CLUSTER
                 CLIENT_TOKEN = token_fetch(aws_account_name: "$AWS_ACCOUNT_NAME", environment_name: "$CLUSTER", vault_url: "$VAULT_URL" )
                //  def getVaultSecretJSON = sh(returnStdout: true, script: "aws secretsmanager get-secret-value --secret-id vault-cicd-approle --region eu-west-2 | jq .SecretString -r") 
                //  getVaultSecret = readJSON text: getVaultSecretJSON
                //  def vaultSecretToken  = getVaultSecret.'secret-id-token'
                //  def vaultRoleId = getVaultSecret.'role-id'
                 
                //  def secretIDToken = sh(script:"set +x; curl -s -k -X POST ${vault_url}/v1/${AWS_ACCOUNT_NAME}/auth/approle/role/cicd-approle-role/secret-id --header 'Content-Type: application/json' --header 'X-Vault-Token: ${vaultSecretToken}' | jq -r '.data.secret_id'", returnStdout: true)
                //  secretIDToken = secretIDToken.trim()
                
                //  def oneTimeToken = sh(script:"set +x; curl -s -k --data '{\"role_id\": \"${vaultRoleId}\",\"secret_id\": \"${secretIDToken}\"}' -X POST ${vault_url}/v1/${AWS_ACCOUNT_NAME}/auth/approle/login  --header 'Content-Type: application/json' | jq -r '.auth.client_token'", returnStdout: true)
                //  oneTimeToken = oneTimeToken.trim()
    
                 sh(script:"set +x; curl -s -k -o credential.json -X GET ${vault_url}/v1/${AWS_ACCOUNT_NAME}/secrets/data/iac-secrets/${clusterName} --header 'Content-Type: application/json' --header 'X-Vault-Token: ${CLIENT_TOKEN}'", returnStdout: true)
                 def readCredential = readJSON file:"credential.json" 
                 credentialData = readCredential.data.data
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
		
		stage("Get OCP Token") {
		    steps {
		        script {
		           sh '''
    		            sh $WORKSPACE/openshift-secret-router/install/get_ocp_token.sh $CLUSTER
		            '''
                   isTokenExist = fileExists file: "$WORKSPACE/octoken"
                    ocToken = readFile(file: "$WORKSPACE/octoken")
		        }
		    }
		}
		
      stage('Load Environment Definition') {
         steps {
             script {
                 def environmentDefinition = readYaml file: "$WORKSPACE/resources/environment-definition.yaml"
                 def clusterInfo = environmentDefinition.environment_definition[clusterName]
                 clusterInfo.each { k,v ->
                    println "ENV_PREFIX: ${k}"
                    def envInfo = clusterInfo[k]
                    println envInfo

                    if(envInfo.openshift) {
                        appDInt = envInfo.openshift.appDRequired == null ? false : envInfo.openshift.appDRequired
                        appDVersion = appDInt ? environmentDefinition.appDAgentVersion : ""
                        imageTag = envInfo.openshift.imageTag == null ? "promote" : envInfo.openshift.imageTag
                        openshiftNamespace = envInfo.openshift.namespace
                    }

                    build(job: "pre-config-per-env",parameters: [
                        string(name: 'ENV_PREFIX', value: k),
                        string(name: 'OPENSHIFT_TOKEN_USERNAME', value: "jenkins"),
                        password(name: 'OPENSHIFT_TOKEN_PASSWORD', value: hudson.util.Secret.fromString(ocToken.trim())),
                        string(name: 'APIGEE_USERNAME', value: "apigee-admin@obcopenldap.obphoenix.co.uk"),
                        password(name: 'APIGEE_PASSWORD', value: hudson.util.Secret.fromString(credentialData.APIGEE_ADMIN_PASSWORD)),
                        string(name: 'CASSANDRA_USERNAME', value: credentialData.DATASTAX_CREDENTIAL_USERNAME),
                        password(name: 'CASSANDRA_PASSWORD', value: hudson.util.Secret.fromString(credentialData.DATASTAX_CREDENTIAL_PASSWORD)),
                        string(name: 'OPENSHIFT_CLUSTER_URL', value: "insecure://${clusterName}-openshift.${AWS_ACCOUNT_NAME}.aws.nbscloud.co.uk:8443"),
                        string(name: 'OPENSHIFT_REGISTRY_URL', value: "docker-registry-default.${clusterName}-openshift-app.${AWS_ACCOUNT_NAME}.aws.nbscloud.co.uk"),
                        string(name: 'OPENSHIFT_NAMESPACE', value: openshiftNamespace),
                        string(name: 'OPENSHIFT_DEPLOY_TAG', value: imageTag),
                        string(name: 'OC_TOOL', value: "OC3.11"),
                        string(name: 'APIGEE_MGMT_SERVER_URL', value: "http://${clusterName}.apigee-mgmtapi.${AWS_ACCOUNT_NAME}.aws.nbscloud.co.uk:8080"),
                        string(name: 'APIGEE_ORG_NAME', value: envInfo.apigee.org_name),
                        string(name: 'APIGEE_ENV_NAME', value: envInfo.apigee.virtual_envs.name),
                        string(name: 'CQL_CONTACT_POINTS', value: "datastax-${clusterName}.${AWS_ACCOUNT_NAME}.aws.nbscloud.co.uk"),
                        string(name: 'CASSANDRA_EXEC_REQUIRED', value: "true"),
                        string(name: 'CASSANDRA_SSL_REQUIRED', value: "FALSE"),
                        string(name: 'CUSTOM_TEMPLATE_NAME', value: "aws"),
                        string(name: 'ENABLE_CUSTOM_TEMPLATE', value: "true"),
                        string(name: 'ENABLE_APPD', value: appDInt.toString()),
                        appDInt ? string(name: 'APPD_AGENT_IMAGE_TAG', value: appDVersion): string(name: 'DUMMY_VAR', value: "DUMMY_VAL")],
                        propagate: false,quietPeriod: 5)
                 }
             }
         }
      }      
   }
   post {
       always {
           cleanWs()
       }
   }
}