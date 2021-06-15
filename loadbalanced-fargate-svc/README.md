# AWS Proton Sample Load-Balanced Web Service Using Amazon ECS and AWS Fargate

This directory contains sample AWS Proton Environment and Service templates for an Amazon ECS load-balanced service running on AWS Fargate, as well as sample specs for creating Proton Environments and Services using the templates. The environment template contains an ECS Cluster and a VPC with two public subnets. The service template contains all the resources required to create an ECS Fargate service behind a load balancer in that environment, as well as sample specs for creating Proton Environments and Services using the templates.

Developers provisioning their services can configure the following properties through their service spec:
* Fargate CPU size
* Fargate memory size
* Number of running containers

If you need application code to run in the service, you can find a sample application here: https://github.com/aws-samples/aws-proton-sample-fargate-service

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
  --region us-west-2 \
  --bucket "proton-cli-templates-${account_id}" \
  --create-bucket-configuration LocationConstraint=us-west-2
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
  --region us-west-2 \
  --pipeline-service-role-arn "arn:aws:iam::${account_id}:role/ProtonServiceRole"
```

Create an AWS CodeStar Connections connection to your application code stored in a GitHub or Bitbucket source code repository.  This connection allows CodePipeline to pull your application source code before building and deploying the code to your Proton service.  To use sample application code, first create a fork of the sample application repository here:
https://github.com/aws-samples/aws-proton-sample-fargate-service

Creating the source code connection must be completed in the CodeStar Connections console:
https://us-west-2.console.aws.amazon.com/codesuite/settings/connections?region=us-west-2

## Register An Environment Template

Register the sample environment template, which contains an ECS Cluster and a VPC with two public subnets.

First, create an environment template, which will contain all of the environment template's versions.

```
aws proton create-environment-template \
  --region us-west-2 \
  --name "public-vpc" \
  --display-name "PublicVPC" \
  --description "VPC with Public Access and ECS Cluster"
```

Now create a version which contains the contents of the sample environment template. Compress the sample template files and register the version:

```
tar -zcvf env-template.tar.gz environment/

aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz --region us-west-2

rm env-template.tar.gz

aws proton create-environment-template-version \
  --region us-west-2 \
  --template-name "public-vpc" \
  --description "Version 2" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=env-template.tar.gz}"
```

Wait for the environment template version to be successfully registered. Use this command to check for registration status:

```
aws proton get-environment-template-version \
  --region us-west-2 \
  --template-name "public-vpc" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the environment template version, making it available for users in your AWS account to create Proton environments.

```
aws proton update-environment-template-version \
  --region us-west-2 \
  --template-name "public-vpc" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Register A Service Template

Register the sample service template, which contains all the resources required to provision an ECS Fargate service behind a load balancer as well as a continuous delivery pipeline using AWS CodePipeline.

First, create the service template.

```
aws proton create-service-template \
  --region us-west-2 \
  --name "lb-fargate-service" \
  --display-name "LoadbalancedFargateService" \
  --description "Fargate Service with an Application Load Balancer"
```

Now create a version which contains the contents of the sample service template. Compress the sample template files and register the version:

```
tar -zcvf svc-template.tar.gz service/

aws s3 cp svc-template.tar.gz s3://proton-cli-templates-${account_id}/svc-template.tar.gz --region us-west-2

rm svc-template.tar.gz

aws proton create-service-template-version \
  --region us-west-2 \
  --template-name "lb-fargate-service" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"public-vpc","majorVersion":"1"}]'
```

Wait for the service template to be successfully registered. Use this command to check for registration status:

```
aws proton get-service-template-version \
  --region us-west-2 \
  --template-name "lb-fargate-service" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the service template version, making it available for users in your AWS account to create Proton services.

```
aws proton update-service-template-version \
  --region us-west-2 \
  --template-name "lb-fargate-service" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Deploy An Environment and Service

With the registered and published environment and service templates, you can now instantiate a Proton environment and service from the templates.

First, deploy a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your AWS account using the Proton service role.

```
aws proton create-environment \
  --region us-west-2 \
  --name "Beta" \
  --template-name public-vpc \
  --template-major-version 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy. Use this command to check for deployment status:

```
aws proton get-environment \
  --region us-west-2 \
  --name "Beta"
```

Then, create a Proton service and deploy it into your Proton environment.  This command reads your service spec at `specs/svc-spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in your AWS account using the Proton service role.  The service will provision a load-balanced ECS service running on Fargate and a CodePipeline pipeline to deploy your application code.

Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```
aws proton create-service \
  --region us-west-2 \
  --name "front-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name lb-fargate-service \
  --spec file://specs/svc-spec.yaml
```

Wait for the service to successfully deploy. Use this command to check for deployment status:

```
aws proton get-service \
  --region us-west-2 \
  --service-name "front-end"
```
