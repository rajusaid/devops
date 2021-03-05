@NonCPS
def parsingTemplate(fileName,fileInstance,mappingVariable,stubFolder) {
    def engine = new org.apache.commons.lang3.text.StrSubstitutor(mappingVariable)
    def templateText = engine.replace(fileInstance)
    writeFile file: "$WORKSPACE/${stubFolder}/$fileName", text: templateText
    engine = null
    templateText = null
}
def executestage = false
pipeline {
    agent {
        label "jenkins-node"
    }

    parameters {
      string (
        name: 'CLUSTER',
        defaultValue: 'nbs-obc-dev',
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
                    checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, 
                    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "ob-pipeline-script"]], 
                    submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
					if(CLUSTER.contains('nbs-obc-dev')){
                        executestage = true
                    }
                }
            }
        }
        
        stage('Apply the Conformance-Suite tool.') {
		    when{
                expression { executestage }
            }
            steps {
                script {
		try{
                    dir("ob-pipeline-script/openshift-template/conformance-suite") {
                        def conformanceRoute = readFile "ob-conformance-route.yaml"
                        def conformanceDC = readFile "ob-conformance-deployment-config.yaml"
						def mappingVariable = [:]
        				mappingVariable.put("CLUSTER_NAME",CLUSTER)
        				mappingVariable.put("AWS_ACCOUNT_NAME",AWS_ACCOUNT_NAME)
                        mappingVariable.put("VERSION",'latest')
        			    parsingTemplate("ob-conformance-route.yaml",conformanceRoute,mappingVariable,'ob-pipeline-script/openshift-template/conformance-suite')
						parsingTemplate("ob-conformance-deployment-config.yaml",conformanceDC,mappingVariable,'ob-pipeline-script/openshift-template/conformance-suite')
						
        				openshift.withCluster(CLUSTER) {
        				    openshift.withProject("nbs-obc-stubs") {
                                openshift.raw('apply', '-f', '.')
        				    }    
        				}
        			}
			}catch(e){
			println e
			}
				
                }
            }
        } 
        stage('Run Nginx'){
            when{ expression{executestage}}
            steps{
                script{
                    dir("ob-pipeline-script/openshift-template/conformance-suite/nginx")
                    {
                        def nginxRoute = readFile "ob-nginx-route.yaml"
						def mappingVariable = [:]
        				mappingVariable.put("CLUSTER_NAME",CLUSTER)
        				mappingVariable.put("AWS_ACCOUNT_NAME",AWS_ACCOUNT_NAME)
        			    parsingTemplate("ob-nginx-route.yaml",nginxRoute,mappingVariable,'ob-pipeline-script/openshift-template/conformance-suite/nginx')
						
                        openshift.withCluster(CLUSTER) {
                                openshift.withProject("nbs-obc-stubs") {
                                    openshift.raw('apply', '-f', '.')
                            }
                        } 
                    }
                }
            }
        }         
    }
}
