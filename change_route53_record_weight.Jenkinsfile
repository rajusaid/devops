pipeline {
agent { label "$CHOOSE_NODE" }

parameters {
	choice (
		name: 'CHOOSE_NODE',
		choices: ['ansible-server', 'ansible-server2'],
		description: 'Executor to run the build on')
	string (
		name: 'HOSTED_ZONE_ID',
		defaultValue: '',
		description: 'Hosted zone id to operate in')
	string (
		name: 'RECORD_NAME',
		defaultValue: '',
		description: 'Record name to change')
	string (
		name: 'SET_IDENTIFIER',
		defaultValue: '',
		description: 'Set identifier of the record e.g nbs-cop-preprod2')   
	string (
		name: 'WEIGHT',
		defaultValue: '0',
		description: 'The weight which is to be set to the record')          
}

stages {
	stage('Update Route 53 Record if it exists') {
		steps {
			script {
				sh '''
					set +x
					echo "============ Getting existing record details ============"
					set -x
					records=$(aws route53 list-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --query "ResourceRecordSets[?SetIdentifier == '$SET_IDENTIFIER']")

					set +x
					if [ "$records" == "[]" ]; then
						 echo "A record with the provided details does not exist, refusing to create a new one. Exiting without taking any actions"
						 exit 0
					fi
					
					TARGET=$(echo "$records" | jq -r '.[].AliasTarget.DNSName')
					TYPE=$(echo "$records" | jq -r '.[].Type')
					EVALUATE_TARGET_HELTH=$(echo "$records" | jq -r '.[].AliasTarget.EvaluateTargetHealth')

					# Target record\'s hosted zone ID. Not the same as the Route 53 Hosted zone
					HOSTED_ZONE=$(echo "$records" | jq -r '.[].AliasTarget.HostedZoneId') 
					
					if [ -z "TARGET" ]; then
					   echo "Something went wrong when trying to retrieve the existing resource record's DNS record entry"
					   exit 1
					fi
					
					if [ -z "$TYPE" ]; then
					   echo "Something went wrong when trying to retrieve the existing resource record's Type field"
					   exit 1
					fi
					
					if [ -z "$EVALUATE_TARGET_HELTH" ]; then
						echo "Something went wrong when trying to retrieve the existing resource record's evaluate target health field"
						exit 1
					fi

					if [ -z "$HOSTED_ZONE" ]; then
						echo "Something went wrong when trying to retrieve the existing resource record's target zone field"
						exit 1
					fi

					echo "============ Performing the change ============"

					# Tab identation is required
					cat > params.json <<-EOF
					{
						\"Comment\": \"Update the record\",
						\"Changes\": [
						{
						\"Action\": \"UPSERT\",
								\"ResourceRecordSet\": {
									\"Name\": \"$RECORD_NAME\",
									\"Type\": \"$TYPE\",
									\"SetIdentifier\": \"$SET_IDENTIFIER\",
									\"Weight\": $WEIGHT,
									\"AliasTarget\": {
										\"DNSName\": \"$TARGET\",
										\"HostedZoneId\": \"$HOSTED_ZONE\",
										\"EvaluateTargetHealth\": $EVALUATE_TARGET_HELTH
									}
								}
							}
						]
					}
					EOF

					if aws  route53 change-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID --change-batch file://params.json ; then
						echo "\n============ The route 53 record was updated successfully ============"
					else
						echo "\n============ The route 53 record update operation failed ============"
					fi
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