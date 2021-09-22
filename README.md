Step 1- aws configure --profile urlshorteneradmin
test aws s3 ls --profile urlshorteneradmin <br /> <br />
Step 2 - git clone https://github.com/mayurbhagia/urlshortenerinfraascode.git
<br />cd urlshortenerinfraascode

Step 3 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-infra --template-file "1_infra.yaml" --capabilities CAPABILITY_IAM


Step 4 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-pinpoint-setup --template-file "2_urlshortener-pinpoint-setup.yaml" --capabilities CAPABILITY_IAM

Goto Pinpoint project -> Settings -> SMS and voice -> SMS Settings -> Edit -> "Default message type"- Transactional, Account Spending Limit - USD 1 from 0 -> Save Changes. Please make sure to raise tickets for increase in SMS monthly spend limit and change from sandbox to production account when setting this for production scale 

If you want to get the SMS delivery think of the solution - https://aws.amazon.com/solutions/implementations/digital-user-engagement-events-database/

Step 5 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-database --template-file "3_urlshortener_database.yaml" --parameter-overrides "VPC=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='VPC'].OutputValue" --output=text)" "PrivateSubnet1=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='PrivateSubnet1'].OutputValue" --output=text)" "PrivateSubnet2=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='PrivateSubnet2'].OutputValue" --output=text)" --capabilities CAPABILITY_IAM

This is multi-AZ database, hence will take 15-20 minutes

Step 6 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-loadbalancer --template-file "4_urlshortener_loadbalancer.yaml" --parameter-overrides "VPC=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='VPC'].OutputValue" --output=text)" "PublicSubnet1=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='PublicSubnet1'].OutputValue" --output=text)" "PublicSubnet2=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='PublicSubnet2'].OutputValue" --output=text)" --capabilities CAPABILITY_IAM

aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-loadbalancer-listenerrule --template-file "4b_urlshortener_loadbalancer-listenerrule.yaml" --parameter-overrides "LBDNSName=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-loadbalancer --query="Stacks[0].Outputs[?OutputKey=='ALBOutput'].OutputValue" --output=text)" "LBListenerARN=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-loadbalancer --query="Stacks[0].Outputs[?OutputKey=='ListenerARN'].OutputValue" --output=text)" "YourDomainName=url.mayurbhagia.online" --capabilities CAPABILITY_IAM

Step 7 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-secretsmanager --template-file "5_urlshortener_secretsmanager.yaml" --parameter-overrides "pinpointprojectid=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-pinpoint-setup --query="Stacks[0].Outputs[?OutputKey=='pinpointprojectid'].OutputValue" --output=text)" "datasourceurl=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-database --query="Stacks[0].Outputs[?OutputKey=='URLshortenerdatabaseendpoint'].OutputValue" --output=text)" --capabilities CAPABILITY_IAM

This has below 8 parameters and secrets configured:
RDS username
RDS password
RDS Endpoint and port
Domain name - this is required to be a short domain owned by the user/enterprise (refer pre-requisites section)
Country Code - Country code for sending SMS for e.g. +91 for India
Sending id - Sender id / Short code received and configured at Telco/Authority portal - this is optional
Keyword - Keyword configured with Telco/Authority - this is optional
URL Retention - How long the short URL needs to be in the database, post which it will be purged


Step 8 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-ecs-cluster-task-and-service --template-file "6_urlshortener_ECS_task_and_service.yaml" --parameter-overrides "VPC=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='VPC'].OutputValue" --output=text)" "PrivateSubnet1=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='PrivateSubnet1'].OutputValue" --output=text)" "PrivateSubnet2=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-infra --query="Stacks[0].Outputs[?OutputKey=='PrivateSubnet2'].OutputValue" --output=text)" "urlshortenertg=$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-loadbalancer --query="Stacks[0].Outputs[?OutputKey=='TargetGroupOutput'].OutputValue" --output=text)" "deploymentregion=us-east-1" --capabilities CAPABILITY_IAM

Please note, this is using the image from dockerhub - mayurbhagia/urlshortenercrossregion.latest and the source code for the same is - https://github.com/mayurbhagia/urlshortener.git. Here i have kept number of tasks as 1, but you can increase and also extend to configure the ECS autoscaling policies

Step 9 - aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-api-gateway-method-1 --template-file "7_urlshortener_api_gateway_method_1.yaml" --parameter-overrides "loadbalancerarn=http://$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-loadbalancer --query="Stacks[0].Outputs[?OutputKey=='ALBOutput'].OutputValue" --output=text)" --capabilities CAPABILITY_IAM

The Method 1 in API is taking two parameters, Long URL to be convereted and Phone number

Step 10 -  aws --profile urlshorteneradmin cloudformation deploy --no-fail-on-empty-changeset --stack-name urlshortener-api-gateway-method-2 --template-file "8_urlshortener_api_gateway_method_2.yaml" --parameter-overrides "loadbalancerarn=http://$(aws --profile urlshorteneradmin cloudformation describe-stacks --stack-name=urlshortener-loadbalancer --query="Stacks[0].Outputs[?OutputKey=='ALBOutput'].OutputValue" --output=text)" --capabilities CAPABILITY_IAM

The Method 2 in API is taking three parameters, Message which has long URL and other content, flag stating whether it has URL or not and Phone number
