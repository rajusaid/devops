pipeline {
    agent { label "ansible-server2"}

    stages {
        stage('Preparation') {
        	steps {
				script {   
					cleanWs()
					checkout([$class: 'GitSCM', branches: [[name: scm.branches[0].name]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
				}
			}
		}

        stage('Backup LDIF') {
			steps {
				script {
					sh "python3 $WORKSPACE/py_modules/ldap_backup/ldapBackup.py"
				}
			}
		}
    }
}