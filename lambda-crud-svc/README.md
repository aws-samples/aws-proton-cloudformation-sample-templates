# AWS Proton Sample CRUD API service Using AWS Lambda

This directory contains sample AWS Proton Environment and Service templates for a CRUD API service built on AWS Lambda, as well as sample specs for creating Proton Environments and Services using the templates. The environment template contains a simple Amazon DynamoDB table. The service template contains all the resources required to provision Lambda-backed APIs as well as a continuous delivery pipeline using AWS CodePipeline.

Developers provisioning their services can configure the following properties through their service spec:
* Lambda memory size
* Lambda function timeout
* Lambda runtime
* A resource to construct a CRUD API around.

If you need application code to run in the service, you can find a sample application here: https://github.com/aws-samples/aws-proton-sample-lambda-crud-service

# Registering and deploying these templates

You can register and deploy these templates by using the AWS Proton console. To do this, you will need to compress the templates using the instructions below, upload them to an S3 bucket, and use the Proton console to register and test them. If you prefer to use the Command Line Interface, follow the instructions below:

## Prerequisites

First, make sure you have the AWS CLI installed and configured. Run the following command to set an environment variable with your account ID:

```
account_id=`aws sts get-caller-identity|jq -r ".Account"`
```

### Configure the AWS CLI

While AWS Proton is in Preview, you will need to manually configure the AWS CLI. The following commands will add the Proton commands to the AWS CLI.

```
aws s3 cp s3://aws-proton-preview-public-files/model/5-19-2021/proton-2020-07-20.normal.json .
aws s3 cp s3://aws-proton-preview-public-files/model/5-19-2021/waiters2.json .
aws configure add-model --service-model file://proton-2020-07-20.normal.json --service-name proton-preview
mv waiters2.json ~/.aws/models/proton-preview/2020-07-20/waiters-2.json
rm proton-2020-07-20.normal.json
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
aws proton-preview update-account-roles \
  --region us-west-2 \
  --account-role-details "pipelineServiceRoleArn=arn:aws:iam::${account_id}:role/ProtonServiceRole"
 ```

Create an AWS CodeStar Connections connection to your application code stored in a GitHub or Bitbucket source code repository.  This connection allows CodePipeline to pull your application source code before building and deploying the code to your Proton service.  To use sample application code, first create a fork of the sample application repository here:
https://github.com/aws-samples/aws-proton-sample-lambda-crud-service

Creating the source code connection must be completed in the CodeStar Connections console:
https://us-west-2.console.aws.amazon.com/codesuite/settings/connections?region=us-west-2

## Register An Environment Template

Register the sample environment template, which contains a simple Amazon DynamoDB table.

First, create an environment template, which will contain all of the environment template's major and minor versions.

```
aws proton-preview create-environment-template \
  --region us-west-2 \
  --template-name "crud-api" \
  --display-name "CRUD Environment" \
  --description "Environment with DDB Table"
```

Then, create a new major version for the `crud-api` environment template.

```
aws proton-preview create-environment-template-major-version \
  --region us-west-2 \
  --template-name "crud-api" \
  --description "Version 1"
```

Now create a minor version which contains the contents of the sample environment template. Compress the sample template files and register the minor version:

```
tar -zcvf env-template.tar.gz environment/

aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz --region us-west-2

rm env-template.tar.gz

aws proton-preview create-environment-template-minor-version \
  --region us-west-2 \
  --template-name "crud-api" \
  --description "Version 1" \
  --major-version-id "1" \
  --source-s3-bucket proton-cli-templates-${account_id} \
  --source-s3-key env-template.tar.gz
```

Wait for the environment template minor version to be successfully registered:

```
aws proton-preview wait environment-template-registration-complete \
  --region us-west-2 \
  --template-name "crud-api" \
  --major-version-id "1" \
  --minor-version-id "0"
```

You can now publish the environment template minor version, making it available for users in your AWS account to create Proton environments.

```
aws proton-preview update-environment-template-minor-version \
  --region us-west-2 \
  --template-name "crud-api" \
  --major-version-id "1" \
  --minor-version-id "0" \
  --status "PUBLISHED"
```

## Register A Service Template

Register the sample service template, which contains all the resources required to provision Lambda-backed APIs as well as a continuous delivery pipeline using AWS CodePipeline.

First, create the service template.

```
aws proton-preview create-service-template \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --display-name "CRUD API Service" \
  --description "CRUD API Service backed by AWS Lambda"
```

Then, create a major version for the `http-api-service` service template and associate it with the `crud-api` environment template created above.

```
aws proton-preview create-service-template-major-version \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --description "Version 1" \
  --compatible-environment-template-major-version-arns arn:aws:proton:us-west-2:${account_id}:environment-template/crud-api:1
```

Now create a minor version which contains the contents of the sample service template. Compress the sample template files and register the minor version:

```
tar -zcvf svc-template.tar.gz service/

aws s3 cp svc-template.tar.gz s3://proton-cli-templates-${account_id}/svc-template.tar.gz --region us-west-2

rm svc-template.tar.gz

aws proton-preview create-service-template-minor-version \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --description "Version 1" \
  --major-version-id "1" \
  --source-s3-bucket proton-cli-templates-${account_id} \
  --source-s3-key svc-template.tar.gz
```

Wait for the service template minor version to be successfully registered:

```
aws proton-preview wait service-template-registration-complete \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --major-version-id "1" \
  --minor-version-id "0"
```

You can now publish the service template minor version, making it available for users in your AWS account to create Proton services.

```
aws proton-preview update-service-template-minor-version \
  --region us-west-2 \
  --template-name "crud-api-service" \
  --major-version-id "1" \
  --minor-version-id "0" \
  --status "PUBLISHED"
```

## Deploy An Environment and Service

With the registered and published environment and service templates, you can now instantiate a Proton environment and service from the templates.

First, deploy a Proton environment. This command reads your environment spec at `specs/env-spec.yaml`, merges it with the environment template created above, and deploys the resources in a CloudFormation stack in your AWS account using the Proton service role.

```
aws proton-preview create-environment \
  --region us-west-2 \
  --environment-name "crud-api-beta" \
  --environment-template-arn arn:aws:proton:us-west-2:${account_id}:environment-template/crud-api \
  --template-major-version-id 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

Wait for the environment to successfully deploy.

```
aws proton-preview wait environment-deployment-complete \
  --region us-west-2 \
  --environment-name "crud-api-beta"
```

Then, create a Proton service and deploy it into your Proton environment.  This command reads your service spec at `specs/svc-spec.yaml`, merges it with the service template created above, and deploys the resources in CloudFormation stacks in your AWS account using the Proton service role.  The service will provision a Lambda-based CRUD API endpoint and a CodePipeline pipeline to deploy your application code.

Fill in your CodeStar Connections connection ID and your source code repository details in this command.

```
aws proton-preview create-service \
  --region us-west-2 \
  --service-name "tasks-front-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version-id 1 \
  --service-template-arn arn:aws:proton:us-west-2:${account_id}:service-template/crud-api-service \
  --spec file://specs/svc-spec.yaml
```

Wait for the service to successfully deploy.

```
aws proton-preview wait service-creation-complete \
  --region us-west-2 \
  --service-name "tasks-front-end"
```

Once the service is created, retrieve the CodePipeline pipeline console URL and the CRUD API endpoint URL for your service.

```
aws proton-preview get-service \
  --region us-west-2 \
  --service-name "tasks-front-end" \
  --query "service.pipeline.outputs" \
  --output text

aws proton-preview get-service-instance \
  --region us-west-2 \
  --service-name "tasks-front-end" \
  --service-instance-name "front-end" \
  --query "serviceInstance.outputs" \
  --output text
```
