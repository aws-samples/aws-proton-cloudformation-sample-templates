# AWS Proton Sample .NET Framework Microservices Using Amazon ECS and AWS Fargate

This directory contains a sample AWS Proton Environment and Service templates for a set of .NET Framework 4.8 microservices deployed to Amazon ECS based using service discovery running on AWS Fargate and a CI/CD pipeline used to deploy updates, as well as sample specs for creating Proton Environments and Services using the templates. All resources deployed are tagged.

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

If you need application code to run in the services:
* https://github.com/aws-quickstart/quickstart-dotnetfx-ecs-cicd

# Registering and deploying these templates

You can register and deploy these templates by using the AWS Proton console. To do this, you will need to compress the templates using the instructions below, upload them to an S3 bucket, and use the Proton console to register and test them. If you prefer to use the Command Line Interface, follow the instructions below:

## Prerequisites

First, make sure you have the AWS CLI installed, and configured. Run the following command to set the AWS region and an `account_id` environment variable:

```bash
export AWS_DEFAULT_REGION=us-west-2
account_id=`aws sts get-caller-identity --query Account --output text`
```

### Configure IAM Role, S3 Bucket, and CodeStar Connections Connection

Before you register your templates and deploy your environments and services, you will need to create an Amazon IAM role so that AWS Proton can manage resources in your AWS account, an Amazon S3 bucket to store your templates, and a CodeStar Connections connection to pull and deploy your application code.

Create the S3 bucket to store your templates:

```
aws s3api create-bucket \
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
  --pipeline-service-role-arn "arn:aws:iam::${account_id}:role/ProtonServiceRole"
```

Create an AWS CodeStar Connections connection to your application code stored in a GitHub or Bitbucket source code repository.  This connection allows CodePipeline to pull your application source code before building and deploying the code to your Proton service. To use sample application code for the public service, first create a fork of the sample application repository here:
https://github.com/aws-quickstart/quickstart-dotnetfx-ecs-cicd

Creating the source code connection must be completed in the CodeStar Connections console:
https://us-west-2.console.aws.amazon.com/codesuite/settings/connections?region=us-west-2

## Register an Environment Template

Register the sample environment template, which contains an ECS Cluster and a VPC with two public subnets.

First, create an environment template, which will contain all of the environment template's versions.

```
aws proton create-environment-template \
  --name "aws-proton-fargate-microservices-dotnetfx" \
  --display-name "aws proton fargate microservices-dotnetfx" \
  --description "Proton Example Dev VPC with Public facing services and with Private backend services on ECS cluster with Fargate compute"
```

Now create a version which contains the contents of the sample environment template. Compress the sample template files and register the version:

```
tar -zcvf env-template.tar.gz environment/

aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz

rm env-template.tar.gz

aws proton create-environment-template-version \
  --template-name "aws-proton-fargate-microservices-dotnetfx" \
  --description "Proton Example Dev Environment Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=env-template.tar.gz}"
```

Wait for the environment template version to be successfully registered.

```
aws proton wait environment-template-version-registered \
  --template-name "aws-proton-fargate-microservices-dotnetfx" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the environment template version, making it available for users in your AWS account to create Proton environments.

```
aws proton update-environment-template-version \
  --template-name "aws-proton-fargate-microservices-dotnetfx" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Register the Service Templates

Register the sample service template, which contains all the resources required to provision an ECS Fargate service behind a load balancer, the private services as well as a continuous delivery pipeline using AWS CodePipeline for each.

First, create the service template.

```
aws proton create-service-template \
  --name "lb-public-fargate-svc-dotnetfx" \
  --display-name "PublicLoadbalancedDotNetFargateService" \
  --description ".NET Framework Windows Fargate Service with an Application Load Balancer"
```

Now create a version which contains the contents of the sample service template. Compress the sample template files and register the version:

```
tar -zcvf svc-public-template.tar.gz service/loadbalanced-public-svc/

aws s3 cp svc-public-template.tar.gz s3://proton-cli-templates-${account_id}/svc-public-template.tar.gz

rm svc-public-template.tar.gz

aws proton create-service-template-version \
  --template-name "lb-public-fargate-svc-dotnetfx" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-public-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"aws-proton-fargate-microservices-dotnetfx","majorVersion":"1"}]'
```

Wait for the service template version to be successfully registered.

```
aws proton wait service-template-version-registered \
  --template-name "lb-public-fargate-svc-dotnetfx" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the Public service template version, making it available for users in your AWS account to create Proton services.

```
aws proton update-service-template-version \
  --template-name "lb-public-fargate-svc-dotnetfx" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

### Second, create the Private service template.

```
aws proton create-service-template \
  --name "private-fargate-svc-dotnetfx" \
  --display-name "PrivateBackendDotNetFargateService" \
  --description ".NET Framework Windows Private Backend Fargate Service"
```

Now create a version which contains the contents of the sample service template. Compress the sample template files and register the version:

```
tar -zcvf svc-private-template.tar.gz service/private-fargate-svc-dotnetfx/

aws s3 cp svc-private-template.tar.gz s3://proton-cli-templates-${account_id}/svc-private-template.tar.gz

rm svc-private-template.tar.gz

aws proton create-service-template-version \
  --template-name "private-fargate-svc-dotnetfx" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-private-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"aws-proton-fargate-microservices-dotnetfx","majorVersion":"1"}]'
```

Wait for the service template version to be successfully registered.

```
aws proton wait service-template-version-registered \
  --template-name "private-fargate-svc-dotnetfx" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the Public service template version, making it available for users in your AWS account to create Proton services.

```
aws proton update-service-template-version \
  --template-name "private-fargate-svc-dotnetfx" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Deploy An Environment

With the registered and published environment template, you can now instantiate a Proton environment from the template.

You can use two different environment provisioning methods when you create environments.

* Create, manage and provision an environment in a single account.

* In a single management account create and manage an environment that is provisioned in another account with environment account connections. For more information, see [Create an environment in one account and provision in another account](https://docs.aws.amazon.com/proton/latest/adminguide/ag-create-env.html#ag-create-env-deploy-other) and [Environment account connections](https://docs.aws.amazon.com/proton/latest/adminguide/ag-env-account-connections.html).

### Create and Provision Environment in a single account

First, deploy a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your AWS account using the Proton service role.

```
aws proton create-environment \
  --name "Beta" \
  --template-name aws-proton-fargate-microservices-dotnetfx \
  --template-major-version 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy.

```
aws proton wait environment-deployed --name Beta
  
aws proton get-environment --name Beta
```
### Create Environment in one account and Provision in another account

First, log into the environment account where you want to provision the environment resources and create the IAM role that Proton will assume to provision resources and manage AWS CloudFormation stacks in your AWS account.
This can also be done from the console. https://docs.aws.amazon.com/proton/latest/adminguide/security_iam_service-role-policy-examples.html#proton-svc-role

```bash
environment_account_id=`your_environment_account_id`

aws iam create-role \
  --role-name ProtonServiceRole \
  --assume-role-policy-document file://./policies/proton-service-assume-policy.json

aws iam attach-role-policy \
  --role-name ProtonServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Then, create and send an environment account connection request to your management account. When the request is accepted, AWS Proton can use the associated IAM role that permits environment resource provisioning in the associated environment account.
You need to specify the environment name that you will use for the environment.

```bash
aws proton create-environment-account-connection \
  --management-account-id ${account_id} \
  --environment-name "Beta" \
  --role-arn arn:aws:iam::${environment_account_id}:role/ProtonServiceRole
  
environment_account_connection_id=`replace_with_the_environment_account_connection_id_returned_above`
```

Log into the management account and accept the environment account connection request from your environment account. This can also be done from the console.

```bash
aws proton accept-environment-account-connection --id ${environment_account_connection_id}
```

Then, create a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your environment AWS account using the Proton service role attached to the environment account connection.

```bash
aws proton create-environment \
  --name "Beta" \
  --template-name aws-proton-fargate-microservices-dotnetfx \
  --template-major-version 1 \
  --environment-account-connection-id ${environment_account_connection_id} \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy. Use the `get` call to check for deployment status:

```bash
aws proton wait environment-deployed  --name Beta
  
aws proton get-environment --name Beta
```

## Deploy A Service

With the registered and published service template and deployed environment, you can now create a Proton service and deploy it into your Proton environment.

This command reads your service spec at `specs/svc-public--spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in the AWS account of the environment.  
The service will provision a Lambda-based CRUD API endpoint and a CodePipeline pipeline to deploy your application code.

Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```
aws proton create-service \
  --name "front-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name lb-public-fargate-svc-dotnetfx \
  --spec file://specs/svc-public-spec.yaml
```

Wait for the service to successfully deploy.

```
aws proton wait service-created --name front-end

aws proton get-service --name front-end
```

And finally, create a Private Proton service and deploy it into your Proton environment.  This command reads your service spec at `specs/svc-private-spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in your AWS account using the Proton service role. The service will provision a private ECS service running on Fargate and a CodePipeline pipeline to deploy your application code.


Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```
aws proton create-service \
  --name "back-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name private-fargate-svc-dotnetfx \
  --spec file://specs/svc-private-spec.yaml
```

Wait for the service to successfully deploy.

```
aws proton wait service-created --name back-end

aws proton get-service --name back-end
```