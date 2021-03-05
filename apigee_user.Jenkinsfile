import groovy.json.JsonOutput;

pipeline {
   agent {
       label "jenkins-node"
   }

    parameters {
      string(name: 'ENV_PREFIX', defaultValue: 'AWS_DEV', description: 'Environment Prefix Ex: AWS_ST')
    }

    
   stages {
      stage("Git Checkout") {
          steps {
              script {
                checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins-github-credentials', url: "$BASE_GIT_URL/$PIPELINE_REPO"]]])
              }
          }
      }

    stage ('Create Apigee User') {      
      steps{
        script{

        def apigeeFile = readYaml file: "$WORKSPACE/resources/apigee-users.yaml"
        def apigeeInfo = apigeeFile.EMAIL
            apigeeInfo.each { x ->
            def apigeeEmail = x.mail
            def apigeeFirstName = x.fname
            def apigeeLastName = x.lname
            def apigeeRole = "orgadmin"
            def apigeePassword = "Apigee123"
            def apigeeDetails='<User><FirstName>'+apigeeFirstName+'</FirstName><LastName>'+apigeeLastName+'</LastName><EmailId>'+apigeeEmail+'</EmailId><Password>'+apigeePassword+'</Password></User>'
            def apigeeDet="<User><FirstName>$apigeeFirstName</FirstName><LastName>$apigeeLastName</LastName><EmailId>$apigeeEmail</EmailId><Password>$apigeePassword</Password></User>"

            // get environment values
            def apigeeBaseUrl = env["${ENV_PREFIX}_APIGEE_MGMT_SERVER_URL"]
            def apigeeOrgName = env["${ENV_PREFIX}_APIGEE_ORG_NAME"]
            def apigeeEnvName = env["${ENV_PREFIX}_APIGEE_ENV_NAME"]

            // get credentials
            def apigeeToken = "${ENV_PREFIX}_APIGEE_TOKEN"
        
            withCredentials([usernamePassword(credentialsId: "${apigeeToken}", usernameVariable: 'APIGEE_ADMIN_USER',passwordVariable:'APIGEE_ADMIN_PASS')]) {

              def userExists = sh(returnStdout: true, script: "curl -k -s -X GET $apigeeBaseUrl/v1/users/$apigeeEmail -u $APIGEE_ADMIN_USER:$APIGEE_ADMIN_PASS -o /dev/null -w '%{http_code}'")


              if( userExists == "200" ) { 
                  println "User $apigeeEmail already exists. User will be listed in Apigee console once the role is assigned"
                  println "Assiging the role to $apigeeFirstName ...."

                  def assignRole = sh(returnStdout: true, script: "curl -k -s -X POST $apigeeBaseUrl/v1/o/$apigeeOrgName/userroles/$apigeeRole/users?id=$apigeeEmail -H 'Content-Type:application/x-www-form-urlencoded' -u $APIGEE_ADMIN_USER:$APIGEE_ADMIN_PASS  -o /dev/null -w '%{http_code}'")
                      println assignRole

                      if ( assignRole == "200" ) {
                            println "Role has been successfully assigned."

                          } else {
                            println "Issue in assigning the role"
                          }

                } else { //if user doesnt exist


                sh """
                curl -k -s -X POST "$apigeeBaseUrl"/v1/users -H 'Content-Type:text/xml' -u "$APIGEE_ADMIN_USER:$APIGEE_ADMIN_PASS" -d "$apigeeDetails" -o /dev/null -w '%{http_code}'
                """

                def newApigeeUser = sh(returnStdout: true, script: "curl -k -s -X GET $apigeeBaseUrl/v1/users/$apigeeEmail -u $APIGEE_ADMIN_USER:$APIGEE_ADMIN_PASS -o /dev/null -w '%{http_code}'")
                
                println newApigeeUser

                if( newApigeeUser == "200" ) { 
                  println "User $apigeeEmail has been sucessfully created. User will be listed in Apigee console once the role is assigned"
                  println "Assiging the role to $apigeeFirstName ...."

                  def assignRole = sh(returnStdout: true, script: "curl -k -s -X POST $apigeeBaseUrl/v1/o/$apigeeOrgName/userroles/$apigeeRole/users?id=$apigeeEmail -H 'Content-Type:application/x-www-form-urlencoded' -u $APIGEE_ADMIN_USER:$APIGEE_ADMIN_PASS  -o /dev/null -w '%{http_code}'")
                      println assignRole

                      if ( assignRole == "200" ) {
                            println "Role has been successfully assigned."

                          } else {
                            println "Issue in assigning the role"
                          }
                } else { 
                  println "Issues creating the user $apigeeEmail"

                }



                }    
              
        }
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
