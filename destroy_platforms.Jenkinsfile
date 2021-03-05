def apigeeSeriveName

pipeline {
	agent { label "jenkins-node" }

	environment {
		PROJECT_NAME = "nbs-obc"
	}

	parameters {		
		string (
			name: 'CLUSTER_NAME',
			defaultValue: 'nbs-obc-test',
			description: 'Environment in which to provision the cluster in')
	}

   	stages {
     	stage ('Cluster Destruction') {
			parallel {
					stage ('Datastax Destruction') {
						steps {
							build job: 'nbs-obc-run-terraform-operations', parameters: [
								string(name: 'CHOOSE_NODE', value: 'ansible-server2'), 
								string(name: 'PROJECT_NAME', value: PROJECT_NAME), 
								string(name: 'ENVIRONMENT_NAME', value: CLUSTER_NAME), 
								string(name: 'SERVICE_NAME', value: 'datastax'), 
								string(name: 'TERRAFORM_OPERATION', value: 'destroy'), 
								booleanParam(name: 'SKIP_VALIDATION', value: true)
							]
						}
					}
					stage ('Apigee Destruction') {
						steps {
							script {
								if(CLUSTER_NAME == "nbs-obc-mctc") {
									apigeeSeriveName = "apigee-12"
								} else {
									apigeeSeriveName = "apigee"
								}								
							}
							build job: 'nbs-obc-run-terraform-operations', parameters: [
								string(name: 'CHOOSE_NODE', value: 'ansible-server2'),
								string(name: 'PROJECT_NAME', value: PROJECT_NAME), 
								string(name: 'ENVIRONMENT_NAME', value: CLUSTER_NAME), 
								string(name: 'SERVICE_NAME', value: apigeeSeriveName ), 
								string(name: 'TERRAFORM_OPERATION', value: 'destroy'), 
								booleanParam(name: 'SKIP_VALIDATION', value: true)
							]
						}
					}
					stage ('OCP Destruction') {
					steps {
						build job: 'nbs-obc-run-terraform-operations', parameters: [
							string(name: 'CHOOSE_NODE', value: 'ansible-server2'), 
							string(name: 'PROJECT_NAME', value: PROJECT_NAME), 
							string(name: 'ENVIRONMENT_NAME', value: CLUSTER_NAME), 
							string(name: 'SERVICE_NAME', value: 'openshift'), 
							string(name: 'TERRAFORM_OPERATION', value: 'destroy'), 
							booleanParam(name: 'SKIP_VALIDATION', value: true)
						]
					}
				}
			}
     	}
   	}
  	post{
		always{
			cleanWs()
		}
  	}
}
