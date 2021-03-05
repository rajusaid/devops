pipeline {
   agent { label "jenkins-node" }

   options {
     	ansiColor('xterm')
    }

    parameters {
      string (
        name: 'APPD_AGENT_VERSION',
        defaultValue: '',
        description: 'AppD Agent Version')
      string (
        name: 'APPD_AGENT_SHA256',
        defaultValue: 'ff0d05764bd072fdda9a879485433127a1d1e82982aad7c071c9687165d827a2',
        description: 'Appd Agent Sha256')
    }

    stages {
      
    stage('Preparation') {
      steps {
        script {
          cleanWs()
          checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
        }
      }
    }

    stage('Download AppD Java Agent from Artifactory') {
        steps {
            script {
                withCredentials([usernamePassword(credentialsId: 'ARTIFACTORY_CI_CREDENTIALS', passwordVariable: 'ARTIFACTORY_PASSWORD', usernameVariable: 'ARTIFACTORY_USERNAME')]) {
                    sh "curl -u ${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD} -O ${ARTIFACTORY_URL}/appdynamics/appd-java-agent/AppServerAgent-${APPD_AGENT_VERSION}.zip"
                    sh "mv AppServerAgent-${APPD_AGENT_VERSION}.zip ./appdynamics/appd-java-agent/"
                }
            }
        }
    }

    stage('Build AppD Java Agent') {
        steps {
            script {
                dir("./appdynamics/appd-java-agent") {
                    sh "chmod +x build.sh"
                    sh "./build.sh ${APPD_AGENT_VERSION} ${APPD_AGENT_SHA256}"
                }
            }
        }
    }
    stage('Push AppD Java Agent to Artifactory') {
        steps {
            script {
                withCredentials([usernamePassword(credentialsId: 'ARTIFACTORY_CI_CREDENTIALS', passwordVariable: 'ARTIFACTORY_PASSWORD', usernameVariable: 'ARTIFACTORY_USERNAME')]) {
                    sh "echo $ARTIFACTORY_PASSWORD | sudo docker login -u $ARTIFACTORY_USERNAME --password-stdin obcomply-docker-dev-local.artifactory.aws.nbscloud.co.uk"
                    sh "sudo docker tag appdynamics/java-agent:${APPD_AGENT_VERSION} obcomply-docker-dev-local.artifactory.aws.nbscloud.co.uk/openbanking/appdynamics-java-agent:${APPD_AGENT_VERSION}"   
                    sh "sudo docker push obcomply-docker-dev-local.artifactory.aws.nbscloud.co.uk/openbanking/appdynamics-java-agent:${APPD_AGENT_VERSION}"   
                }                
            }
        }
    }

   }
}