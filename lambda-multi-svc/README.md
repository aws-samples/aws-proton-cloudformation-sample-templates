# AWS Proton Sample Multi Service

This directory contains sample AWS Proton Environment and Service templates to showcase how you can leverage AWS Proton Environments to create shared resources across multiple Services. The environment template contains a simple Amazon DynamoDB table and an S3 Bucket. Then there are two service templates. One service template creates a simple CRUD API service backed by Lambdas and an API gateway and includes an AWS Code Pipeline for continuos delivery. The second service template creates a data processing service that consumes data from an API, pushes that data into a kinesis stream which is then subsequently consumed by another Lambda and then pushes that data into a firehose that ends up in the S3 Bucket configured by the environment template, again offering continuous delivery via AWS CodePipeline.

Developers provisioning their services can configure the following properties through their service spec:
* Lambda memory size
* Lambda function timeout
* Lambda runtime
* A resource to construct a CRUD API around.

If you need application code to run in the services:

* For the crud service: https://github.com/aws-samples/aws-proton-sample-lambda-crud-service
* For the data processing service: https://github.com/aws-samples/aws-proton-sample-lambda-data-processing-service


# Registering and deploying these templates

You can register and deploy these templates by using the AWS Proton console. To do this, you will need to compress the templates using the instructions below, upload them to an S3 bucket, and use the Proton console to register and test them. If you prefer to use the Command Line Interface, follow the instructions below:

## Prerequisites

First, make sure you have the AWS CLI installed and configured. Run the following command to set an environment variable with your account ID:

```bash
account_id=`aws sts get-caller-identity --query Account --output text`
```

### Configure IAM Role, S3 Bucket, and CodeStar Connections Connection

Before you register your templates and deploy your environments and services, you will need to create an Amazon IAM role so that AWS Proton can manage resources in your AWS account, an Amazon S3 bucket to store your templates, and a CodeStar Connections connection to pull and deploy your application code.

Create the S3 bucket to store your templates:

```bash
aws s3api create-bucket \
  --region us-west-2 \
  --bucket "proton-cli-templates-${account_id}" \
  --create-bucket-configuration LocationConstraint=us-west-2
```

Create the IAM role that Proton will assume to provision resources and manage AWS CloudFormation stacks in your AWS account.
This can also be done from the console. https://docs.aws.amazon.com/proton/latest/adminguide/ag-setting-up-iam.html#setting-up-cicd 

```bash
aws iam create-role \
  --role-name ProtonServiceRole \
  --assume-role-policy-document file://./policies/proton-service-assume-policy.json

aws iam attach-role-policy \
  --role-name ProtonServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Then, allow Proton to use that role to provision resources for your services' continuous delivery pipelines:

```bash
aws proton update-account-settings \
  --region us-west-2 \
  --pipeline-service-role-arn "arn:aws:iam::${account_id}:role/ProtonServiceRole"
```

Create an AWS CodeStar Connections connection to your application code stored in a GitHub or Bitbucket source code repository.  This connection allows CodePipeline to pull your application source code before building and deploying the code to your Proton service.  To use sample application code, first create a fork of the sample application repository here:
https://github.com/aws-samples/aws-proton-sample-lambda-crud-service

Creating the source code connection must be completed in the CodeStar Connections console:
https://us-west-2.console.aws.amazon.com/codesuite/settings/connections?region=us-west-2

## Register An Environment Template

Register the sample environment template, which contains a simple Amazon DynamoDB table.

First, create an environment template, which will contain all of the environment template's versions.

```bash
aws proton create-environment-template \
  --region us-west-2 \
  --name "multi-svc-env" \
  --display-name "Multi Service Environment" \
  --description "Environment with DDB Table & S3 Bucket"
```

Now create a version which contains the contents of the sample environment template. Compress the sample template files and register the version:

```bash
tar -zcvf env-template.tar.gz lambda-env/

aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz --region us-west-2

rm env-template.tar.gz

aws proton create-environment-template-version \
  --region us-west-2 \
  --template-name "multi-svc-env" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=env-template.tar.gz}"
```

Wait for the environment template version to be successfully registered. You can use this command to see the registration status

```bash
aws proton wait environment-template-version-registered \
  --region us-west-2 \
  --template-name "multi-svc-env" \
  --major-version "1" \
  --minor-version "0"
  
aws proton get-environment-template-version \
  --region us-west-2 \
  --template-name "multi-svc-env" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the environment template version, making it available for users in your AWS account to create Proton environments.

```bash
aws proton update-environment-template-version \
  --region us-west-2 \
  --template-name "multi-svc-env" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Register the CRUD Service Template

Register the sample service template, which contains all the resources required to provision Lambda-backed APIs as well as a continuous delivery pipeline using AWS CodePipeline.

First, create the service template.

```bash
aws proton create-service-template \
  --region us-west-2 \
  --name "crud-api-service" \
  --display-name "CRUD API Service" \
  --description "CRUD API Service backed by AWS Lambda"
```

Now create a version which contains the contents of the sample service template. Compress the sample template files and register the version:

```bash
tar -zcvf svc-template.tar.gz service-crud/

aws s3 cp svc-template.tar.gz s3://proton-cli-templates-${account_id}/svc-template.tar.gz --region us-west-2

rm svc-template.tar.gz

aws proton create-service-template-version \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"multi-svc-env","majorVersion":"1"}]'
```

Wait for the service template version to be successfully registered. You can use this command to see the registration status

```bash
aws proton wait service-template-version-registered \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --major-version "1" \
  --minor-version "0"
  
aws proton get-service-template-version \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --major-version "1" \
  --minor-version "0"
```

You can now publish the service template version, making it available for users in your AWS account to create Proton services.

```bash
aws proton update-service-template-version \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

## Register the Data Processing Service Template

Repeat all of the steps above but change template-name to `data-processing-service` (or whatever you want to call it) and create a tar by running `tar -zcvf svc-data-processing-template.tar.gz service-data-processing/`.

## Deploy An Environment

With the registered and published environment template, you can now instantiate a Proton environment from the template.

You can use two different environment provisioning methods when you create environments.

* Create, manage and provision an environment in a single account.

* In a single management account create and manage an environment that is provisioned in another account with environment account connections. For more information, see [Create an environment in one account and provision in another account](https://docs.aws.amazon.com/proton/latest/adminguide/ag-create-env.html#ag-create-env-deploy-other) and [Environment account connections](https://docs.aws.amazon.com/proton/latest/adminguide/ag-env-account-connections.html).

### Create and Provision Environment in a single account

First, deploy a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your AWS account using the Proton service role.

```bash
aws proton create-environment \
  --region us-west-2 \
  --name "multi-svc-beta" \
  --template-name multi-svc-env \
  --template-major-version 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy. Use the `get` call to check for deployment status:

```bash
aws proton wait environment-deployed \
  --region us-west-2 \
  --name "multi-svc-beta"
  
aws proton get-environment \
  --region us-west-2 \
  --name "multi-svc-beta"
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
  --region us-west-2 \
  --management-account-id ${account_id} \
  --environment-name "multi-svc-beta" \
  --role-arn arn:aws:iam::${environment_account_id}:role/ProtonServiceRole
  
environment_account_connection_id=`replace_with_the_environment_account_connection_id_returned_above`
```

Log into the management account and accept the environment account connection request from your environment account. This can also be done from the console. 

```bash
aws proton accept-environment-account-connection \
  --region us-west-2 \
  --id ${environment_account_connection_id}
```

Then, create a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your environment AWS account using the Proton service role attached to the environment account connection.

```bash
aws proton create-environment \
  --region us-west-2 \
  --name "multi-svc-beta" \
  --template-name multi-svc-env \
  --template-major-version 1 \
  --environment-account-connection-id ${environment_account_connection_id} \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy. Use the `get` call to check for deployment status:

```bash
aws proton wait environment-deployed \
  --region us-west-2 \
  --name "multi-svc-beta"
  
aws proton get-environment \
  --region us-west-2 \
  --name "multi-svc-beta"
```

## Deploy A Service

With the registered and published service template and deployed environment, you can now create a Proton service and deploy it into your Proton environment.

This command reads your service spec at `specs/svc-spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in the AWS account of the environment.  
The service will provision a Lambda-based CRUD API endpoint and a CodePipeline pipeline to deploy your application code.

Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```bash
aws proton create-service \
  --region us-west-2 \
  --name "tasks-front-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name crud-api-service \
  --spec file://specs/svc-spec.yaml
```

Wait for the service to successfully deploy. Use this call to check for deployment status:

```bash
aws proton wait service-created \
  --region us-west-2 \
  --name "tasks-front-end"

aws proton get-service \
  --region us-west-2 \
  --name "tasks-front-end"
```
