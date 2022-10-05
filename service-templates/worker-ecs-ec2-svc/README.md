## Description

This template is compatible with the [ecs-ec2-env](../../environment-templates/ecs-ec2-env) template. It creates an ECS service running on a Self-managed cluster of EC2 hosts, that allows you to implement asynchronous service-to-service communication with [pub/sub architectures](https://aws.amazon.com/pub-sub-messaging/). The microservices in your application can publish events to an [Amazon SNS topics](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) that can then be consumed by a "Worker Service". A Worker Service is composed of: 

1. One or more [Amazon SQS queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) to process notifications published to the topics, as well as [dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) to handle failures.
2. An Amazon ECS service running on EC2 that has permission to poll the SQS queues and process the messages asynchronously.

Service properties like desired task count, task size (cpu/memory units), and docker image URL can be specified through the service input parameters. 

The template also provisions a CodePipeline based pipeline to pull your application source code before building and deploying it to the Proton service. To use sample application code, please fork the sample code repository [aws-proton-sample-services](https://github.com/aws-samples/aws-proton-sample-services). By default, the template deploys a [python application](https://github.com/aws-samples/aws-proton-sample-services/tree/main/ecs-worker) that polls the SQS queue for messages, and writes  the message body to CloudWatch logs.

## Architecture

### Public Subnet
![worker-ecs-ec2-public-srv](../../images/worker-ecs-ec2-public-srv.png)

### Private Subnet
![worker-ecs-ec2-private-srv](../../images/worker-ecs-ec2-private-srv.png)

## Parameters

### Service Inputs

1. desired_count: The default number of Fargate tasks you want running
2. task_size: The size of the task you want to run
3. image: The name/url of the container image

### Pipeline Inputs

1. service_dir: Source directory for the service
2. dockerfile: The location of the Dockerfile to build
3. unit_test_command: The command to run to unit test the application code
4. environment_account_ids: The environment account ids for service instances using cross account environment

## Test
This worker service can be tested by deploying the [ecs-worker](https://github.com/aws-samples/aws-proton-sample-services/tree/main/ecs-worker) application that reads the SQS queue and writes the message to CloudWatch logs. We then can deploy a [scheduled-ecs-ec2-svc](../scheduled-ecs-ec2-svc/) that runs [ecs-ping-sns](https://github.com/aws-samples/aws-proton-sample-services/tree/main/ecs-ping-sns) application that sends a random message to the shared SNS topic, every 5 minutes. Expected data in CloudWatch logs:
```
INFO: The message body: {
  "Type" : "Notification",
  "MessageId" : "605944d3-5636-5e1c-8a77-1e3aaa0dc93b",
  "TopicArn" : "arn:aws:sns:us-east-2:XXXXXXXXXXXX:AWSProton-ecs-ec2-env-pro-cloudformation--MPDTYJAPBFYGPYF-ping",
  "Message" : "Hello! Message srptt sent at time Monday, May 02 2022, 15:34:19",
  "Timestamp" : "2022-05-02T15:34:19.387Z",
  "SignatureVersion" : "1",
  "Signature" : "WNHwbHUbTdc7yvJEmY1S9cvGUE0JMuP/PR6GNYe1p7bnyYGcJ7wB3QU3W/5264/VkPN+eMm+FOkW/PZAQr36Tww8TwPMr21xoSZyXfYoyvyk1FecS2i3IMmgRrYBQAi9BbBIhZO+2zBD35640rKAPakvPbNjhV+SNfeF3cPBezlSAgREUd+TovsmI+78h8AIa+dmUaZHFKCFCFmhOo+ovLZGQoLw+H4ow/YofFyzZCr/jNx4iHiI7K15YQ6TPky0S+4xpxMhjJD6vQ2XR75cnZpjKgQ6ip+uTXC4eKYE6mRHW/JBeriwKMv6TaQ3UatiJheyFJ28WRQxAZWeOW1k4g==",
  "SigningCertURL" : "https://sns.us-east-2.amazonaws.com/SimpleNotificationService-7ff5318490ec183fbaddaa2a969abfda.pem",
  "UnsubscribeURL" : "https://sns.us-east-2.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:us-east-2:XXXXXXXXXXXX:AWSProton-ecs-ec2-env-pro-cloudformation--MPDTYJAPBFYGPYF-ping:445f255f-c616-4852-aa76-b30a40864848"
}

INFO: Deleting message from the queue...

INFO: Received and deleted message(s) from https://sqs.us-east-2.amazonaws.com/XXXXXXXXXXXX/AWSProton-worker-ecs-ec2-worker-ecs-ec2-c-ServiceEcsProcessingQueue-XWL8D7ptF4mP with message
```

## Security

See [CONTRIBUTING](../../CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](../../LICENSE) file.
