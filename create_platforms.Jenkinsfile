pipeline {
   	agent { label "jenkins-node" }

	environment {
		PROJECT_NAME = "nbs-obc"
		AWS_ACCOUNT_NAME = "nbs-obc-dev"
	}

	parameters {
		string (
			name: 'CLUSTER_NAME',
			defaultValue: 'nbs-obc-test',
			description: 'Environment in which to provision the cluster in')
		string (
			name: 'ENV_PREFIX',
			defaultValue: 'AWS_DT',
			description: 'Env prefix')
	}

	stages {
		// Infrastucture and platform provisioning
		stage ('Cluster Provisioning') {
			parallel {
				stage ('Apigee Provisioning') {
					steps {
						build job: 'nbs-obc-apigee-provision', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME), 
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME), 
							string(name: 'ARTIFACTORY_DEPENDENCY_REPO_PATH', value: 'obcomply-gen-rel-local/ob-iaac/'),
							booleanParam(name: 'DESTROY_ANY_EXISTING_CLUSTER', value: true)
						]
					}
				}
				stage ('OCP Provisioning') {
					steps {
							build job: 'nbs-obc-openshift-provision', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME), 
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME), 
							booleanParam(name: 'DESTROY_ANY_EXISTING_CLUSTER', value: true)
						]
					}
				}
				stage ('Datastax Provisioning') {
					steps {
						build job: 'nbs-obc-datastax-provision', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME), 
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME), 
							booleanParam(name: 'DESTROY_ANY_EXISTING_CLUSTER', value: true)
						]
					}
				}
			}
		}
		// OCP env preparation
		stage ('OCP Cluster Provisioning') {
			steps {
				build job: 'OpenBanking_Platform/nbs-obc-openshift-cluster-env-preparation', parameters: [
					string(name: 'CLUSTER', value: CLUSTER_NAME), 
					string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME)
				]
			}
		}
		// Environment setup
		stage ('Environment Config') {
			parallel {
				stage ('Apigee VENV Creation') {
					steps {
						build job: 'OpenBanking_Platform/nbs-obc-apigee-virtual-env-creation', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME),
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME)
						]
					}
				}
				stage ('OCP VENV Creation') {
					steps {
						build job: 'OpenBanking_Platform/nbs-obc-openshift-virtual-env-creation', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME), 
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME)
						]
					}
				}
				stage ('DSE VEnv Creation') {
					steps {
						build job: 'nbs-obc-datastax-virtual-env-creation', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME), 
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME)
						]
					}
				}
				stage ('Install Logging Config in OpenShift') {
					steps {
						build job: 'nbs-obc-dev-logging-efk', parameters: [
							string(name: 'CLUSTER', value: CLUSTER_NAME),
							string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME)
						]
					}
				}
			}
		}


		// Twistlock Integration
		stage ('OCP Twistlock Integration') {
			steps {
				build job: 'OpenBanking_Platform/nbs-obc-twistlock-integration', parameters: [
					string(name: 'CHOOSE_NODE', value: 'ansible-server'),
					string(name: 'ENV_NAME', value: 'twistlock'),
					string(name: 'CLUSTER', value: CLUSTER_NAME)
				]
			}
		}			

		// Get cluster config detail
		stage ('Get Cluster Config Detail') {
			steps {
				build job: 'get-cluster-config-detail', parameters: [
					string(name: 'CLUSTER', value: CLUSTER_NAME),
					string(name: 'AWS_ACCOUNT_NAME', value: AWS_ACCOUNT_NAME)
				]
			}
		}
		
		// Wiremock proxy
		stage ('Wiremock Proxy') {
			steps {
				build job: 'wiremock-proxy', parameters: [
					string(name: 'CLUSTER', value: CLUSTER_NAME),
					string(name: 'AWS_ACCOUNT_NAME', value: 'nbs-obc-dev')
				]
			}
		}

		//Create user accounts for Apigee
		stage ('Apigee User Creation') {
			steps {
				build job: 'nbs-obc-apigee-user-creation', parameters: [
					string(name: 'ENV_PREFIX', value: "$ENV_PREFIX")
				]
			}
		}

		// Execute Baseline
		stage ('Execute Baseline') {
			steps {
				build job: 'execute-baseline', parameters: [
					string(name: 'CLUSTER', value: CLUSTER_NAME),
					string(name: 'AWS_ACCOUNT_NAME', value: 'nbs-obc-dev')
				]
			}
		}	
	}

	post{
    	always{
      		cleanWs()
    	}
  	}
}