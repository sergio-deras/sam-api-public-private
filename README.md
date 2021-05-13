# AWS-SAM-Express

A simple demo to show the integration of AWS Serverless Application Model (SAM) and an NodeJS Express application,
that will deploy to an API gateway and Lambda.

## Setup

### Pre-reqs
* Python 3.6 (or 2.7) or higher or SAM CLI already installed and configured
* AWS CLI
* Node 8.10 or higher
* Docker if testing locally


### PYTHON VENV
```bash
cd sam-express-sample
python3 -m venv venv
source venv/bin/activate
```

### SAM
```
pip install aws-sam-cli
```

### App Dependencies
```bash
npm install
```
---
## Test locally

**Emulate API Gateway locally (docker image, hot-reloading for interpreted languages)**
```bash
sam local start-api
```
If the previous command ran successfully you should now be able to hit the following local endpoint to invoke your function `http://localhost:3000/`. _The first time this request is made, it will download the Docker image, which may take a long time. Even when this is done, subsequent requests will check if this is the latest. 

```bash
curl -v http://localhost:3000/
curl -v http://localhost:3000/users
curl -v http://localhost:3000/users/1/events -d '{"foo": {"bar": "quux"}}' \
-H 'Content-Type: application/json'
```

**Emulate Lambdas locally (use awscli, boto3 or other sdk)**
```bash
sam local start-lambda
aws lambda invoke --function-name "SocialEventsFunction" --endpoint-url "http://127.0.0.1:3001" --no-verify-ssl out.txt
```

**Emulate a single call to a Lambdas locally**
```bash
sam local invoke "SocialEventsFunction"
sam local invoke "SocialEventsFunction" -e event.json
```

**Generate sample payloads from different services (s3, sqs, sns, dynamodb, kinesis, and others)**
```bash
sam local generate-event
```
---
## Deploy Public Version

### Pre-reqs

* AWS CLI already configured with at least PowerUser permission

### Build

```bash
aws sam build
```

### Deploy 
__First time__
```bash
sam deploy --guided
```

__Next time__
```bash
sam deploy --stack-name <stack-name>
```
The default stack-name is sam-app, this name will be used in the following commands.
You need to reference the stack-name because there are two stacks in the folder.

> Reference: [Serverless Application Model (SAM) HOWTO Guide](https://github.com/awslabs/serverless-application-model/blob/master/HOWTO.md) 

After deployment is complete you can run the following command to retrieve the API Gateway Endpoint URL:

```bash
aws cloudformation describe-stacks \
    --stack-name sam-app \
    --query 'Stacks[].Outputs'
```

After deployment is complete you can run the following command to retrieve the API Gateway Endpoint URL:

```bash
aws cloudformation describe-stacks \
    --stack-name sam-app \
    --query 'Stacks[].Outputs'
```

You can also retrieve the API URI with:
```bash
aws cloudformation describe-stacks --stack-name sam-app --query 'Stacks[].Outputs[1].OutputValue' --profile at-user
```

Test
```bash
curl -i <API_URI>
```

### Destroy
We will deploy the private version next, I recommend that you destroy this deployment, you can do it later. This will delete the CloudFormation the stack and all its resources.
```bash
aws cloudformation delete-stack --stack-name sam-app  
```


## Private API Gateway

For an API gateway accessible through one or more subnets of a single VPC.

![Private API](https://docs.aws.amazon.com/apigateway/latest/developerguide/images/apigateway-private-api-accessing-api.png)


TODO**
#### Requirements

- A VPC, you should know the VPC Id.
- One more Subnet Ids in the VPC - these will be managed by the VPC Endpoint this template creates.
- One security group Id in the VPC - this template will create a security group that allows this security group ingress through port 443.

#### Packaging and deployment

Create your bucket if you don't have one:

```bash
aws s3 mb s3://BUCKET_NAME
```

Next, run the following command to package our Lambda function to S3:

```bash
sam package \
    --template-file template-private-api.yaml \
    --output-template-file template-private-api-packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME
```

Next, the following command will create a Cloudformation Stack and deploy your SAM resources.

sam deploy --template-file template-ref.yaml --stack-name sam-app-network

```bash
sam deploy \
    --template-file template-private-api-packaged.yaml \
    --stack-name sam-app-private \
    --parameter-overrides \
    VpcIdParameter=<VPC Id> \
    VpcAllowedSecurityGroupIdParameter=<Security Group Id> \
    VpcEndpointSubnetIdsParameter=<Subnet 1 Id>,<Subnet 2 Id> \
    --capabilities CAPABILITY_IAM
```

After deployment is complete you can run the following command to retrieve the API Gateway Endpoint URL:

```bash
aws cloudformation describe-stacks \
    --stack-name sam-app \
    --query 'Stacks[].Outputs'
```

If your VPC allows private DNS, then this API will be accessible through an EC2 instance in a managed subnet and allowed security group. If your VPC does not allow private DNS, you can look up the API URL from the VPC Endpoint this template creates.

```bash
aws cloudformation ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<VPC Id>
```

Depending on the number of subnets managed by the endpoint, the last ``DnsEntries.DnsName`` will be the public DNS.
The API can be accessed by the instance in the allowed subnet and security group:
```
curl -v https://vpce-<VPCE Id>-<AWS AVAILABILITY ZONE>.execute-api.us-east-1.vpce.amazonaws.com/Prod/ \
-H 'Host: <REST API Id>.execute-api.<AWS REGION>.amazonaws.com'
```


## Appendix

### AWS CLI commands

AWS CLI commands to package, deploy and describe outputs defined within the cloudformation stack:

```bash
sam package \
    --template-file template.yaml \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME

sam deploy \
    --template-file packaged.yaml \
    --stack-name sam-app \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides MyParameterSample=MySampleValue

aws cloudformation describe-stacks \
    --stack-name sam-app --query 'Stacks[].Outputs'
```

**NOTE**: Alternatively this could be part of package.json scripts section.

* [AWS Serverless Application Repository](https://aws.amazon.com/serverless/serverlessrepo/)
