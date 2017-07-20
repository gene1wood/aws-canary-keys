# Deploy Infrastructure

This is done once to setup the infrastructure for canary-keys

* Create SNS Topic to receive alerts of triggered canary-keys
* Create DynamoDB table to store IAM user/key to deployed location mappings. This is what will associate a given IAM user and their canary-key with a description of where that canary-key will be deployed so that when the canary-key triggers, the incident responders know where the canary-key was located and what system is likely compromised.
* Create Lambda IAM role
    * Disable IAM API Keys
    * Publish to SNS
* Create Lambda function
    * Triggered by CloudWatch event of API call that matches Event Pattern
    * We don't need to determine if the API key used to make the API call was a canary-key because the CloudWatch Rule filters for us
    * Disable API key
    * Alert to SNS
* Create Lambda function resource policies [to grant CloudWatch Events permission to invoke it](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/auth-and-access-control-cwe.html)
* Create IAM Policy for canary
    * Allow actions for products which are [supported by CloudWatch API Call Events](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/EventTypes.html#api_event_type)
    * Disallow actions that change canary infrastructure
    * Allow read actions which aren't supported by API Call Events if you're ok with the attacker seeing the account
    * Disallow read actions that show the canary infrastructure
* Create CloudWatch Events Rule which filters for API Call Events that were made by one of the canary-keys. At this point since we're creating the infrastructure and don't have any canary-keys, this list is empty and as a result *no* API Call Events will match the rule
* Create CloudWatch Events Target which binds the CloudWatch Events Rule to the Lambda function ARN

# Generate Canary Key

This is run each time a user wants to create a new canary-key keypair

* Prompt user for where they intend to deploy the canary-key they are generating.
* Create IAM canary user
* Apply canary IAM policy to the user
* Create IAM access key for the canary user
* Update the canary-key CloudWatch Events Rule to trigger on this new canary-key (adding to existing canary-keys in the rule)
* Write to the DynamoDB table the new canary username and/or access key and the intended canary-key deployment location gathered earlier
* Emit to the SNS topic the details of this new canary-key (but not the secret access key)
* Return the new canary-key keypair to the user so they can deploy them