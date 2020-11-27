# Load Balanced Fargate Service

This repository contains templates and specs for admins to create Environment and Service templates in AWS Proton. The environment contains an ECS Cluster and a VPC with two public subnets. The service template contains the all the resources required to create a Fargate Service in that environment.

Developers provisioning their services can configure the services:
* Fargate cpu size
* Fargate memory size
* Service count

# Registering and deploying these templates

You can register and deploy these templates by using the AWS Proton console. To do this, you will need to compress the templates using the instructions below, upload them to an S3 bucket, and use the Proton console to register and test them. If you prefer to use the Command Line Interface, follow the instructions below:

## Prerequisites

First, make sure you have the AWS CLI installed, and configured. Run the following command to set an environment variable with your account ID:

```
account_id=`aws sts get-caller-identity|jq -r ".Account"`
```

### Configure the AWS CLI

While AWS Proton is in preview, you'll have to manually configure the AWS CLI. The following commands will add the Proton commands to the AWS CLI.

```
aws s3 cp s3://aws-proton-preview-public-files/model/proton-2020-07-20.normal.json .
aws s3 cp s3://aws-proton-preview-public-files/model/waiters2.json .
aws configure add-model --service-model file://proton-2020-07-20.normal.json --service-name proton-preview
mv waiters2.json ~/.aws/models/proton-preview/2020-07-20/waiters-2.json
rm proton-2020-07-20.normal.json
```

### Configure AWS Role and S3 Bucket

Before we register and deploy our environments and services, we need to create a few IAM roles so that AWS Proton can create resources in our accounts - as well as an S3 bucket to store our templates.

Create the S3 bucket to store our templates:

```
aws s3api create-bucket --bucket "proton-cli-templates-${account_id}" --region us-east-1
```

Create the role that Proton will assume to manage resources in your AWS account.

```
aws iam create-role --role-name ProtonServiceRole --assume-role-policy-document file://./policies/proton-service-assume-policy.json
aws iam attach-role-policy --role-name ProtonServiceRole --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Finally, let's allow Proton to use that role for managing Pipelines:

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  update-account-roles \
  --account-role-details "pipelineServiceRoleArn=arn:aws:iam::${account_id}:role/ProtonServiceRole"
 ```

## Register An Environment Template

The first thing we need to do is register the environment templates.

First, let's create an environment template - this will contain all of our environment template major/minor versions.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-environment-template \
  --template-name "public-vpc" \
  --display-name "PublicVPC" \
  --description "VPC with Public Access and ECS Cluster"
```

Now, let's create a new major version for our `public-vpc` environment template.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-environment-template-major-version \
  --template-name "public-vpc" \
  --description "Version 1"
```

Now that we have our major version defined, we can create a minor version which contains the contents of our environment template. Let's tar our environment templates and set up our minor version:

```
tar -zcvf env-template.tar.gz environment/ && aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz && rm env-template.tar.gz

aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-environment-template-minor-version \
  --template-name "public-vpc" \
  --description "Version 2" \
  --major-version-id "1" \
  --source-s3-bucket proton-cli-templates-${account_id} \
  --source-s3-key env-template.tar.gz
```

Now let's wait for the environment template minor version to be registered:

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  wait environment-template-registration-complete \
  --template-name "public-vpc" \
  --major-version-id "1" \
  --minor-version-id "0"
```

If we're happy with our template, we can "publish" it, making it available for developers to register services against.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  update-environment-template-minor-version \
  --template-name "public-vpc" \
  --major-version-id "1" \
  --minor-version-id "0" \
  --status "PUBLISHED"
```


## Register A Service Template

The service template creation experience is basically the same. We'll create a service template, register a major version, and then upload our template as a minor version.

Let's create our service template.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-service-template \
  --template-name "lb-fargate-service" \
  --display-name "LoadbalancedFargateService" \
  --description "Fargate Service with an Application Load Balancer"
```

Now, let's create a new major version for our `lb-fargate-service` service template - and we'll associate it with the environment template we created above.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-service-template-major-version \
  --template-name "lb-fargate-service" \
  --description "Version 1" \
  --compatible-environment-template-major-version-arns arn:aws:proton:us-west-2:${account_id}:environment-template/public-vpc:1
```

Now that we have our major version defined, we can create a minor version which contains the contents of our service template. Let's tar our service templates and set up our minor version:

```
tar -zcvf svc-template.tar.gz service/ && aws s3 cp svc-template.tar.gz s3://proton-cli-templates-${account_id}/svc-template.tar.gz && rm svc-template.tar.gz

aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-service-template-minor-version \
  --template-name "lb-fargate-service" \
  --description "Version 1" \
  --major-version-id "1" \
  --source-s3-bucket proton-cli-templates-${account_id} \
  --source-s3-key svc-template.tar.gz
```

Now let's wait for the service template minor version to be registered:

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  wait service-template-registration-complete \
  --template-name "lb-fargate-service" \
  --major-version-id "1" \
  --minor-version-id "0"
```

If we're happy with our template, we can "publish" it, making it available for developers to use.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  update-service-template-minor-version \
  --template-name "lb-fargate-service" \
  --major-version-id "1" \
  --minor-version-id "0" \
  --status "PUBLISHED"
```

## Deploy Environment and Service

Now that we've got our environment and service templates registered and publish, we can instantiate an environment and deploy an actual service into it.

Let's deploy our environment. This command will read your environment spec at `specs/env-spec.yaml`, merges it with the environment template we created above, and deploys it to your account using the Proton roles we setup at the start of this guide.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-environment \
  --environment-name "Beta" \
  --environment-template-arn arn:aws:proton:us-west-2:${account_id}:environment-template/public-vpc \
  --template-major-version-id 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

After we create our environment, let's wait for it to get deployed.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  wait environment-deployment-complete \
  --environment-name "Beta"
```

Now let's do the same thing to create our service and pipeline. We'll provide a codestar connection, repository branch and repo in this example. If you don't have those handy, you can use the dummy values below - but your pipeline won't work.

```
aws proton-preview \
  --endpoint-url https://proton.us-west-2.amazonaws.com \
  --region us-west-2 \
  create-service \
  --service-name "front-end" \
  --branch "main" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/YOUR-CODESTAR-CONNECTION \
  --repository-id "YOUROWNER/YOURREPO" \
  --template-major-version-id 1 \
  --service-template-arn arn:aws:proton:us-west-2:{account_id}:service-template/lb-fargate-service \
  --spec file://specs/svc-spec.yaml
```
