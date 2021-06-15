# AWS Proton Sample Microservices Using Amazon ECS and AWS Fargate

This directory contains a sample AWS Proton Environment and Service templates for a set of Amazon ECS based microservices using service discovery running on AWS Fargate, as well as sample specs for creating Proton Environments and Services using the templates. All resources deployed are tagged.

The environment template deploys:

- a VPC
- an Internet Gateway
- 2 Nat Gateways across 2 Availability Zones
- 2 private subnets across 2 Availability Zones
- 2 public subnets across 2 Availability Zones
- an ECS Cluster
- a private namespace for service discovery

The service templates contains all the resources required to create a public ECS Fargate service behind a load balancer and a private ECS Fargate service in that environment. It also provides sample specs for creating Proton Environments and Services using the templates.

Developers provisioning their services can configure the following properties through their service spec:

- Fargate CPU size
- Fargate memory size
- Number of running containers
- Choose private or public subnets to run the loadbalanced service or the private service, accessed via service discovery
- Service name to register and use for service discovery

# Registering and deploying these templates

You can register and deploy these templates by using the AWS Proton console. To do this, you will need to compress the templates using the instructions below, upload them to an S3 bucket, and use the Proton console to register and test them. If you prefer to use the Command Line Interface, follow the instructions below:

## Prerequisites

First, make sure you have the AWS CLI installed, and configured. Run the following command to set an environment variable with your account ID:

```bash
account_id=`aws sts get-caller-identity --query Account --output text`
```

### Configure IAM Role, S3 Bucket, and CodeStar Connections Connection

Before you register your templates and deploy your environments and services, you will need to create an Amazon IAM role so that AWS Proton can manage resources in your AWS account, an Amazon S3 bucket to store your templates, and a CodeStar Connections connection to pull and deploy your application code.

Create the S3 bucket to store your templates:

```
aws s3api create-bucket \
  --region us-east-2 \
  --bucket "proton-cli-templates-${account_id}" \
  --create-bucket-configuration LocationConstraint=us-east-2
```

Create the IAM role that Proton will assume to provision resources and manage AWS CloudFormation stacks in your AWS account.

```
aws iam create-role \
  --role-name ProtonServiceRole \
  --assume-role-policy-document file://./policies/proton-service-assume-policy.json

aws iam attach-role-policy \
  --role-name ProtonServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Then, allow Proton to use that role to provision resources for your services' continuous delivery pipelines:

```
aws proton update-account-settings \
  --region us-east-2 \
  --pipeline-service-role-arn "arn:aws:iam::${account_id}:role/ProtonServiceRole"
```

Create an AWS CodeStar Connections connection to your application code stored in a GitHub or Bitbucket source code repository.  This connection allows CodePipeline to pull your application source code before building and deploying the code to your Proton service. To use sample application code for the public service, first create a fork of the sample application repository here:
https://github.com/aws-samples/aws-proton-sample-fargate-service

Creating the source code connection must be completed in the CodeStar Connections console:
https://us-east-2.console.aws.amazon.com/codesuite/settings/connections?region=us-east-2

## Register an Environment Template

Register the sample environment template, which contains an ECS Cluster and a VPC with two public subnets.

First, create an environment template, which will contain all of the environment template's versions.

```
aws proton \
  --region us-east-2 \
  create-environment-template \
  --name "aws-proton-fargate-microservices" \
  --display-name "aws proton fargate microservices" \
  --description "Proton Example Dev VPC with Public facing services and with Private backend services on ECS cluster with Fargate compute"
```

Now create a version which contains the contents of the sample environment template. Compress the sample template files and register the version:

```
tar -zcvf env-template.tar.gz environment/

aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz --region us-east-2

rm env-template.tar.gz

aws proton \
  --region us-east-2 \
  create-environment-template-version \
  --name "aws-proton-fargate-microservices" \
  --description "Proton Example Dev Environment Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=env-template.tar.gz}"
```

Wait for the environment template version to be successfully registered. Use this command to verify status

```
aws proton get-environment-template-version \
  --region us-east-2 \
  --template-name "aws-proton-fargate-microservices" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the environment template version, making it available for users in your AWS account to create Proton environments.

```
aws proton \
  --region us-east-2 \
  update-environment-template-version \
  --name "aws-proton-fargate-microservices" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Register the Service Templates

Register the sample services templates, which contains all the resources required to provision an ECS Fargate services behind a load balancer, the private services as well as a continuous delivery pipeline using AWS CodePipeline for each.

### First, create the Public service template.

```
aws proton \
  --region us-east-2 \
  create-service-template \
  --name "lb-public-fargate-svc" \
  --display-name "PublicLoadbalancedFargateService" \
  --description "Fargate Service with an Application Load Balancer"
```

Now create a version which contains the contents of the sample service template. Compress the sample template files and register the version:

```
tar -zcvf svc-private-template.tar.gz service/loadbalanced-public-svc/

aws s3 cp svc-private-template.tar.gz s3://proton-cli-templates-${account_id}/svc-private-template.tar.gz

rm svc-private-template.tar.gz

aws proton \
  --region us-east-2 \
  create-service-template-version \
  --name "lb-public-fargate-svc" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-private-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"aws-proton-fargate-microservices","majorVersion":"1"}]'
```

Wait for the service template version to be successfully registered. Use this command to verify status

```
aws proton get-service-template-version \
  --region us-east-2 \
  --template-name "lb-public-fargate-svc" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the Public service template version, making it available for users in your AWS account to create Proton services.

```
aws proton \
  --region us-east-2 \
  update-service-template-version \
  --name "lb-public-fargate-svc" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

### Second, create the Private service template.

```
aws proton \
  --region us-east-2 \
  create-service-template \
  --name "private-fargate-svc" \
  --display-name "PrivateBackendFargateService" \
  --description "Private Backend Fargate Service"
```

Now create a version which contains the contents of the sample service template. Compress the sample template files and register the version:

```
tar -zcvf svc-private-template.tar.gz service/private-fargate-svc/

aws s3 cp svc-private-template.tar.gz s3://proton-cli-templates-${account_id}/svc-private-template.tar.gz

rm svc-private-template.tar.gz

aws proton \
  --region us-east-2 \
  create-service-template-version \
  --name "private-fargate-svc" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-private-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"aws-proton-fargate-microservices","majorVersion":"1"}]'
```

Wait for the service template version to be successfully registered. Use this command to verify status

```
aws proton \
  --region us-east-2 \
  get-service-template \
  --name "private-fargate-svc" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the Public service template version, making it available for users in your AWS account to create Proton services.

```
aws proton \
  --region us-east-2 \
  update-service-template-version \
  --template-name "private-fargate-svc" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Deploy An Environment and Services

With the registered and published environment and service templates, you can now instantiate a Proton environment and service from the templates.

First, deploy a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your AWS account using the Proton service role.

```
aws proton create-environment \
  --region us-east-2 \
  --name "Beta" \
  --template-name aws-proton-fargate-microservices \
  --template-major-version 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy. Use this command to verify deployment status:

```
aws proton get-environment \
  --region us-east-2 \
  --name "Beta"
```

Then, create a Public Proton service and deploy it into your Proton environment.  This command reads your service spec at `specs/svc-public-spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in your AWS account using the Proton service role.  The service will provision a load-balanced ECS service running on Fargate and a CodePipeline pipeline to deploy your application code.

Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```
aws proton create-service \
  --region us-east-2 \
  --name "front-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-east-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name lb-public-fargate-svc \
  --spec file://specs/svc-public-spec.yaml
```

Wait for the service to successfully deploy. Use this command to verify deployment status

```
aws proton get-service \
  --region us-east-2 \
  --name "front-end"
```

And finally, create a Private Proton service and deploy it into your Proton environment.  This command reads your service spec at `specs/svc-private-spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in your AWS account using the Proton service role. The service will provision a private ECS service running on Fargate and a CodePipeline pipeline to deploy your application code.

Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```
aws proton create-service \
  --region us-east-2 \
  --name "back-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-east-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name private-fargate-svc \
  --spec file://specs/svc-private-spec.yaml
```

Wait for the service to successfully deploy. Use this command to verify deployment status:

```
aws proton get-service \
  --region us-east-2 \
  --name "back-end"
```
