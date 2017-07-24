# Infrastructure

This is done once to setup the infrastructure for canary-keys. This could most safely be done in a dedicated AWS account but could also be deployed in an existing AWS account. The risks of deploying this in an existing AWS account is that an attacker could
* Make API read calls to AWS allowing them to see your resources in the account for minutes at a time before the CloudTrail event triggers disabling of the API key
* Make a single (or however many calls can be made before the Lambda function completes) API write call to AWS before triggering disabling of the API key via CloudWatch API Call Events 

If this is deployed in a dedicated account these risks are mitigated by not having resources in the account unrelated to canary-keys

## Deploy

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
* Setup the fast response/limited view trigger to disable the compromised API key. This uses CloudWatch Events triggering on API Call Events. There is a very small delay between when an API call is made and when the lambda function will trigger, however API Call Events in CloudWatch only cover [non-read API calls and only calls to specific products](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/EventTypes.html#api_event_type) 
  * Create CloudWatch Events Rule which filters for API Call Events that were made by one of the canary-keys. At this point since we're creating the infrastructure and don't have any canary-keys, this list is empty and as a result *no* API Call Events will match the rule
  * Create CloudWatch Events Target which binds the CloudWatch Events Rule to the Lambda function ARN
* Setup the slow response/comprehensive view trigger to disable the compromised API key. This uses CloudTrail logs which have a long delay between the API call and when the lambda function will trigger, however it covers many many more API calls than CloudWatch API Call Events, for example read-only API calls. CloudWatch events are created for not only successful API calls by the attacker but also denied API calls by the attacker (for example read only calls or non-read calls on products not yet supported by CloudWatch API Call Events)
  * Create a CloudWatch Log Group
  * [Create an IAM Role](http://docs.aws.amazon.com/awscloudtrail/latest/userguide/send-cloudtrail-events-to-cloudwatch-logs.html#send-cloudtrail-events-to-cloudwatch-logs-cli-create-role) which the CloudTrail service can assume which will [grant CloudTrail permission to write to the designated CloudWatch Log Group](http://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-required-policy-for-cloudwatch-logs.html)
  * [Create a CloudTrail](http://docs.aws.amazon.com/awscloudtrail/latest/userguide/send-cloudtrail-events-to-cloudwatch-logs.html#send-cloudtrail-events-to-cloudwatch-logs-cli-update-trail) that is configured to send to the CloudWatch Log Group created above using the IAM Role created above.

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

# Potential as a Public Service

If this were deployed in a dedicated AWS account, it potentially could be opened up as a public service. That would involve

* Establishing a mapping of undo actions for every non-read action in the AWS API for a given product. For example, a mapping that shows that the opposite of `ec2:CreateSecurityGroup` is `ec2:DeleteSecurityGroup`.
  * This mapping would be done only for the products which one suspects an attacker may be interested in.
  * Initially, only doing `ec2` would make good sense and then later expanding to other products as time permits
* Deploying a web app which allows users to
  * submit their email address
  * have an api key pair and IAM user generated
  * have an SNS topic dedicated to that single API key generated
  * get subscribed to this dedicated SNS topic
  * upon SNS topic subscription confirmation, the API key pair is conveyed to the user
* Constraining the IAM permissions of the generated users to be
  * read permissions which exclude the canary-keys infrastructure and users
  * write permissions to only the products for which an established "undo mapping" has been created
* Extend the lambda function to not only disable the IAM user and api key pair when they are used but also to trigger scanning of the last N minutes of CloudTrail logs before the disabling occurs for all actions taken by that user, and using the "undo mapping" to undo each action the attacker did.

For this to work, exploration into how to prevent an attacker from abusing the service would be needed. For example an attacker could use the canary-keys service to
* Generate an api key
* Make a call to AWS
* API key is blocked
* Generate a second api key
* Make a second call to AWS
* second API key is blocked
* repeat until CloudTrail catches up and resources start getting "undone"

It would also make sense to allow users to subscribe other non-email subscriptions to the SNS topic to allow for their own integrations. This could be accomplished if for example the SNS topic name contained the public portion of the API key to prevent users other than the user that generated it from subscribing to it (via security by obscurity)