pipeline {
   agent { label "$CHOOSE_NODE" }

   options {
     ansiColor('xterm')
   }

   parameters {
        choice(
            name: 'CHOOSE_NODE',
            choices: "ansible-server\nansible-server2",
            description: 'Executor to run the build on')
       choice(
            name: 'PROJECT_NAME',
            choices: "nbs-obc",
            description: 'Terraform project')
       choice(
            name: 'VAULT_BASE_NAMESPACE',
            choices: "nbs-obc-dev",
            description: 'The root vault namespace operating in. Only relevant if deploying elasticache')
        choice(
            name: 'ENVIRONMENT_NAME',
            choices: ['nbs-obc-dev','nbs-obc-test','nbs-obc-mctc','nbs-obc-sandbox','nbs-obc-pt'],
            description: 'Environment to deploy in within the prject')
       choice(
            name: 'SERVICE_NAME',
            choices: "datastax\napigee\napigee-12\nopenshift\nansible-server\nalarms-sns\nalerting\nelk\ncloudhsm\nsftp\njenkins\nelasticache\naws-elk\nsonarqube\nldap",
            description: 'Infrastructure of the service to deploy')
      choice(
            name: 'TERRAFORM_OPERATION',
            choices: "plan\nplan-destroy\nshow-state\nshow-outputs\nvalidate\ndeploy\ndestroy\ndestroy-backend-config-artifacts",
            description: "plan shows a list of changes that will be made\nplan-deploy shows a list of changes that will be made if deploy is executed\nplan-destroy shows the list of resources which will be destroyed if destroy is executed\nshow-state shows the current state\nshow-outputs shows the outputs section of the deployment\nvalidate validates terraform syntax\ndeploy creates or updated terraform resources\ndestroy destroys all resources\ndestroy-backend-config-artifacts destroys the DynamoDB associated statefile lock")
       booleanParam(
            defaultValue: false,
            name: 'SKIP_VALIDATION',
            description: 'Skip the interactive prompt which prompts for terraform operation confirmation. Only relevant for apply/destroy')
   }

   stages {

     stage ('Validate the Terraform Operation') {
        when {
            expression { SKIP_VALIDATION == 'false' && (TERRAFORM_OPERATION == 'deploy' || TERRAFORM_OPERATION == 'destroy') }
        }

        steps {
           dir("terraform/") {
              sh '''
                  echo "\033[31m================= Generating an execution plan =================\033[0m"
                  if [ ${TERRAFORM_OPERATION} = "destroy" ]; then
                    set -x
                    make plan-destroy service=${SERVICE_NAME} project=${PROJECT_NAME} environment=${ENVIRONMENT_NAME}
                  else
                    set -x
                    make plan service=${SERVICE_NAME} project=${PROJECT_NAME} environment=${ENVIRONMENT_NAME}
                  fi

              '''
           }

         /* script {
            def USER_INPUT = input(
              message: "\033[31mPERFORM '${TERRAFORM_OPERATION}' FOR PROJECT '${PROJECT_NAME}' AND ENVIRONMENT '${ENVIRONMENT_NAME}' FOR ${SERVICE_NAME} as per the execution plan?\033[0m")

          }*/
       }
     }

     stage ('Perform the Selected Terraform Operation') {
        steps {
          dir("terraform/") {
              sh '''
                  make ${TERRAFORM_OPERATION} service=${SERVICE_NAME} project=${PROJECT_NAME} environment=${ENVIRONMENT_NAME}
              '''
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
