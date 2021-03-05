def environmentDefinition
def clusterInfo

def convertTextToYaml(releaseFile) {
  def affected = false
  componentInYamlFormat=[:]
  parentComponent= []
  affectParentComponent=[] 
  releaseFile.split("\n").each {
      componentInArray=it.split(" ")
      componentList= [:]
          if(componentInArray[0].matches("ob-(.*)")) {
              if(!affected) {
                componentList.put("componentName",componentInArray[0].trim())
                componentList.put("componentVer",componentInArray[1].trim())
                parentComponent.add(componentList)
              } else {
                componentList.put("componentName",componentInArray[0].trim())
                componentList.put("componentVer",componentInArray[1].trim())
                affectParentComponent.add(componentList)                      
              }
          } else {
             affected= true
          }
      }
  componentInYamlFormat.put("COMPONENTS",parentComponent)
  componentInYamlFormat.put("AFFECTED_COMPONENTS",affectParentComponent)
  return componentInYamlFormat
}

pipeline {
    agent { label "jenkins-node" }

    options {
        ansiColor('xterm')
    }

    parameters {
        string (
            name: 'CLUSTER',
            defaultValue: 'nbs-obc-dev' ,
            description: 'OCP Cluster')
        string (
              name: 'AWS_ACCOUNT_NAME',
              defaultValue: 'nbs-obc-dev',
              description: 'The name of the aws account which is same as vault namespace')
         string (
            name: 'PIPELINE_BRANCH',
            defaultValue: 'master' ,
            description: 'release-10')
    }

    stages {
        stage('Preparation') {
            steps {
                script {
                    cleanWs()
                    checkout([$class: 'GitSCM', branches: [[name: scm.branches[0].name]], doGenerateSubmoduleConfigurations: false, 
                    extensions: [], 
                    submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
                    environmentDefinition= readYaml file: "$WORKSPACE/resources/environment-definition.yaml"
                    clusterInfo = environmentDefinition.environment_definition[CLUSTER]
                }
            }
        }

		stage ('Execute Baseline and STUB') {
			parallel {
				stage ('Execute BASELINE') {
                    steps {
                        script {
                            clusterInfo.each { k,v ->
                                def envInfo = clusterInfo[k]
                                if(envInfo.baselineVersion) {
                                    println "BASELINE Execution ::: ENVIRONMENT: ${k} ::: ${envInfo}"
                                    build(job: "OpenBanking_Baseline/DeployBaseline",parameters: [string(name: 'ENV_PREFIX', value: k),string(name: 'BASELINE_TAG', value: envInfo.baselineVersion),string(name: 'RELEASE_LIST', value: 'OBIE_COMMON_COMPONENT,OBIE_R2_COMPONENT,OBIE_R4_1_COMPONENT,OBIE_R3_COMPONENT'), string(name: 'COMPONENT_TYPE', value: 'API,MICROSERVICE'), string(name: 'APP_CONFIG', value: 'APIGEE_PRE_CONFIG,APIGEE_POST_CONFIG,MICROSERVICE_CONFIG'), string(name: 'STUB_LIST', value: 'AUTH_UI_STUB,HSM_STUB'), string(name: 'RELEASE_TYPE', value: 'FULL'), string(name: 'DELTA_DEPLOYMENT', value: ''),string(name: 'RTP_EXECUTION', value: 'YES'),string(name: 'JMETER_RUN', value: 'NO'),string(name: 'PIPELINE_BRANCH', value: 'release-10')],propagate: false,quietPeriod: 5)
                                }
                            }
                        }
                    }                    
                }
				stage ('Execute STUB') {
                    steps {
                        script {
                            String key = null;
                            iterator = clusterInfo.keySet().iterator()
                            if(iterator.hasNext()){
                                key = iterator.next();
                            }
                            def envInfo = clusterInfo[key]
                            println "STUB Execution ::: ENVIRONMENT: ${key} ::: ${envInfo}"
                            checkout([$class: 'GitSCM',branches: [[name: "${envInfo.baselineVersion}"]],doGenerateSubmoduleConfigurations: false,
                                extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "ob-baseline-release-${envInfo.baselineVersion}"]],
                                gitTool: 'Default',
                                submoduleCfg: [],userRemoteConfigs: [[url: "$BASE_GIT_URL/ob-baseline-release",credentialsId: 'jenkins-github-credentials']]])

                                def _stubFile = readFile file:"ob-baseline-release-${envInfo.baselineVersion}/OB-Stubs.txt"
                                def stubList = convertTextToYaml(_stubFile)
                                def stubComponentList = stubList.COMPONENTS
                                def stubVersion

                                stubComponentList.each { component ->
                                    if(component['componentName'] == "ob-soapui-mocked-nem") {
                                        stubVersion = component['componentVer']
                                    }
                                }
                            build(job: "OpenBanking_Baseline/stubs-execution/full-stub-deploy-to-ocp",parameters: [string(name: 'GIT_REF', value: stubVersion),string(name: 'CLUSTER', value: CLUSTER)],propagate: false,quietPeriod: 5)
                        }
                    }                    
                }                
            }
        }                     
    }
}
