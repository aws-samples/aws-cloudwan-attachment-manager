# Attachment Management to  AWS Cloud WAN



## Table of Content

1. [Prerequisites](#Prerequisites)
2. [Deployment Steps](#deployment-steps)
3. [Deployment Validation](#deployment-validation)
4. [Contributing](#contributing)
5. [Authors](#authors)



## Prerequisites

### Operating System

The deployment method for this guide was tested on an Ubuntu 22.04 Operating System. It should run on any new and supported version of Mac OS X, Linux or Windows, assuming you can install the required packages.

In this guide we cover launching the solution using [CloudFormation](https://aws.amazon.com/cloudformation/) and [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) to do the packaging and pushing the lambdas. The easiest way to install SAM is through [brew](https://brew.sh/). Here's how you could setup your environment on an Ubuntu 22.04 host.

```
# Update and install basic tools
apt update && apt upgrade -y
apt install -y build-essential procps curl file git

# Install brew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"

# Install AWS CLI and SAM
brew install aws-sam-cli awscli
```


[Terraform](https://www.terraform.io/) (or [OpenTofu](https://opentofu.org/)) and [Cloud Development Kit for Terraform (CDK-TF)](https://developer.hashicorp.com/terraform/cdktf) implementations are also available in this repository, but will not be covered in this example.


### AWS account requirements

The following AWS account pre-requisites should be ensured:

1. Modify the AWS Cloud WAN Core network policy (i.e. segments and association method);
2. Create a [Service Control Policy (SCP)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) to prevent user accounts from using the segment association tag;
3. Create an [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) role in the AWS Management Account which allows the querying of AWS Account Tags from the Central Network account;
4. User accounts are tagged with the appropriate route segment / domain tag, as part of the account vending process;

Let's go over each of them, one by one.

#### 1. Modify the AWS Cloud WAN Core network policy (i.e. segments and association method)

First, you need to have a running AWS Cloud WAN Core network with a few pre-requisites configured, including event monitoring. A policy requires one or more segments. In a typical environment, you might have:

- Your business specific segments, e.g. production, staging, testing, wan, infrastructure, etc
- A last-hop return segment containing all destination routes, i.e. every attachment propagation - normally aggregating your east-west inspection VPCs (let’s call it the ```fullreturn``` segment).

Additionally, as part of this solution, you will need to have extra segments, the ```quarantine``` and ```quarantineroutes```:

- ```quarantine``` segment: Into which any untagged segment will be associated; this prevents any kind of communication
- ```quarantineroutes``` segment: Which will be used to learn any routes being propagated from the quarantine VPC segments


The following code snippet shows the declaration of these segments as part of the Cloud WAN Network Policy document.


```
{
  "version": "2021.12",
  "segments": [
    {
      "isolate-attachments": true,
      "name": "quarantine",
      "require-attachment-acceptance": false
    },
    {
      "isolate-attachments": true,
      "name": "quarantineroutes",
      "require-attachment-acceptance": false
    },
    ...
  ],
  "segment-actions": [
    {
      "action": "share",
      "mode": "attachment-route",
      "segment": "quarantine",
      "share-with": [ "quarantineroutes" ]
    },
    ...
  ],
  ...
}
```

The next pre-requisite in the Cloud WAN network policy will be specifying the association method. This will only contain two attachment policies to manage the admission of attachments to the correct segments:

- Most preferred policy, where the route-domain tag value will be used to select the segment for the attachment to be associated with;
- Least preferred policy, where in the absence of the route-domain tag, attachments will be associated with the quarantine segment;

The following code snippet shows the declaration of the association method in the Cloud WAN Network Policy document.


```
{
  ...
  "attachment-policies": [
    {
      "action": {
        "association-method": "tag",
        "tag-value-of-key": "route-domain"
      },
      "conditions": [
        { "type": "any" }
      ],
      "rule-number": 10
    },
    {
      "action": {
        "association-method": "constant",
        "segment": "quarantine"
      },
      "conditions": [
        { "type": "any" }
      ],
      "rule-number": 20
    }
  ],
  ...
}
```


#### 2. Create a Service Control Policy (SCP) to prevent user accounts from using the segment association tag

The next pre-requisite is to make sure that principals in the AWS User account cannot use the ```route-domain``` tag when interacting with the Cloud WAN Core network attachment. Users should be prevented from specifying segment membership metadata. The following SCP example can be used to enforce this pre-requisite and should be applied within the AWS Management Account and for every Organization Unit which is not centrally managed by the infrastructure team


```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAttachmentTags1",
      "Effect": "Deny",
      "Action": [
        "networkmanager:TagResource",
        "networkmanager:CreateVpcAttachment"
      ],
      "Resource": "arn:aws:networkmanager:*",
      "Condition": {
        "ForAllValues:StringEquals": { "aws:TagKeys": [ "route-domain" ] }
      }
    }
  ]
}
```

Note: Depending on the Infrastructure as Code (IaC) tool you are using, your users may need to mark the tag configuration of the network attachments to be ignored (e.g. if using Terraform you will need to use a ```lifecycle``` statement) to prevent attempts of tag overwrite and API errors due to the SCP.


#### 3. Create an AWS Identity and Access Management (IAM) role in the AWS Management Account which allows the querying of AWS Account Tags from the Central Network account

The next pre-requisite, is to have an IAM role deployed in the AWS Management account which your Network Account can assume. This role will only allow you to query AWS Account Tags in the [AWS Organizations](https://aws.amazon.com/organizations/) API for the solution to discover the target segment for VPCs created within the account.

Here's an example of the Trust Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {s
        "AWS": "arn:aws:iam::$NETWORK_ACCOUNT_ID:root"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
```

And here are the permissions for this role:


```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "organizations:Describe*",
        "organizations:List*"
      ],
      "Effect": "Allow",
      "Resource": "*",
      "Sid": "DescribeOrgAccounts"
    }
  ]
}
```


#### 4. User accounts are tagged with the appropriate route segment / domain tag, as part of the account vending process

Finally, you need to ensure that your user accounts are tagged with the appropriate ```route-domain``` tag as part of the account vending process (for example: Production accounts would have a ```route-domain: production``` tag, matching an existing ```production``` segment)


### Supported Regions (if applicable)

This solution is composed of two types of stacks, with the following regional requirements:

* Network Manager Event Processor Stack: needs to be deployed in Oregon Region (us-west-2);
* Attachment Manager Stacks: can be deployed on any region.


## Deployment Steps

1. Deploy the 'Network Manager Event Processor' stack:


```
cd source/network-manager-events/cloudformation/

# Build the lambda and deploy the cloudformation stack
sam build && sam deploy \
  --resolve-s3 \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --stack-name network-manager-events \
  --region us-west-2 \
  --parameter-overrides \
      ParameterKey=Name,ParameterValue="network-manager-events"

# Let's grab the outputs
NETWORK_MANAGER_STACK_OUTPUT=$(aws cloudformation describe-stacks \
  --stack-name network-manager-events \
  --region us-west-2 \
  --query 'Stacks[0].Outputs')
SNS_TOPIC_ARN=$(echo $NETWORK_MANAGER_STACK_OUTPUT | jq -r '.[] | select(.OutputKey=="SnsNetworkEventsArn") | .OutputValue')

cd -
```


2. Deploy the 'Attachment Manager' stack in the main region (don't forget to populate the variables below):

```
# Set the variables with your correct values
GLOBAL_NETWORK_ID="<insert Network Manager Global Network id here>"
ATTACHMENT_MANAGER_NAME="cloudwan-attachment-manager"
AWS_ACCOUNT_READER_ROLE_ARN="<insert arn of IAM Role to read the account tags (aws pre-requisite 3)>"
CORE_NETWORK_ARN="<insert Cloud WAN Core Network arn here>"
FULL_RETURN_TABLE="fullreturn"


cd source/attachment-manager/cloudformation/

aws cloudformation validate-template \
  --template-body file://template.yml \
  --region $MAIN_AWS_REGION

# Copy the vpc segment address map to be packaged with the lambda
cp vpc_segment_address_map.yml ../lambda/attachment_manager

# Build the lambda and deploy the cloudformation stack
sam build && sam deploy \
  --resolve-s3 \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --stack-name cloudwan-attachment-manager \
  --region $MAIN_AWS_REGION \
  --parameter-overrides \
      ParameterKey=Name,ParameterValue=${ATTACHMENT_MANAGER_NAME} \
      ParameterKey=AwsAccountReaderRoleArn,ParameterValue=${AWS_ACCOUNT_READER_ROLE_ARN} \
      ParameterKey=NetworkEventsSnsTopicArn,ParameterValue=${SNS_TOPIC_ARN} \
      ParameterKey=GlobalNetworkId,ParameterValue=${GLOBAL_NETWORK_ID} \
      ParameterKey=CoreNetworkArn,ParameterValue=${CORE_NETWORK_ARN} \
      ParameterKey=FullReturnTable,ParameterValue=${FULL_RETURN_TABLE}
```


3. Deploy the 'Attachment Manager' stack in the redundant region (optional):

```
sam build && sam deploy \
  --resolve-s3 \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --stack-name cloudwan-attachment-manager-secondary \
  --region $REDUNDANT_AWS_REGION \
  --parameter-overrides \
      ParameterKey=Name,ParameterValue=${ATTACHMENT_MANAGER_NAME} \
      ParameterKey=AwsAccountReaderRoleArn,ParameterValue=${AWS_ACCOUNT_READER_ROLE_ARN} \
      ParameterKey=NetworkEventsSnsTopicArn,ParameterValue=${SNS_TOPIC_ARN} \
      ParameterKey=GlobalNetworkId,ParameterValue=${GLOBAL_NETWORK_ID} \
      ParameterKey=CoreNetworkArn,ParameterValue=${CORE_NETWORK_ARN} \
      ParameterKey=FullReturnTable,ParameterValue=${FULL_RETURN_TABLE} \
      ParameterKey=SqsEventsDelaySeconds,ParameterValue="15"
```


## Deployment Validation


1. Verify the 'Network Manager Event Processor' stack deployment output from the SAM execution:

```
...
Successfully created/updated stack - network-manager-events in us-west-2
```


2. Verify the 'Attachment Manager' stack deployment output from the SAM execution, in the main region:

```
...
Successfully created/updated stack - cloudwan-attachment-manager in eu-west-1
```


3. Verify the 'Attachment Manager' stack deployment output from the SAM execution, in the redundant region (optional):

```
...
Successfully created/updated stack - cloudwan-attachment-manager-secondary in us-east-1
```


## Cleanup

To remove the solution, you can execute the following SAM CLI commands:

1. Delete Attachment Manager Stack in the main region

```
sam delete --no-prompts \
  --stack-name cloudwan-attachment-manager \
  --region eu-west-1
```

2. Delete Attachment Manager Stack in the secondary region

```
sam delete --no-prompts \
  --stack-name cloudwan-attachment-manager-secondary \
  --region us-east-1
```

3. Delete the Network Manager Events Stack in us-west-2

```
sam delete --no-prompts \
  --stack-name network-manager-events \
  --region us-west-2
```


To remove the SAM bootstrap, you can go to the console for each of the regions, empty all the relevant buckets from the S3 console, and finally delete the CloudFormation stacks with name aws-sam-cli-managed-default.




## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.


## Authors

- João Rodrigues, Senior Cloud Infrastructure Architect, AWS
- Srivalsan Mannoor Sudhagar, Cloud Infrastructure Architect, AWS


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

