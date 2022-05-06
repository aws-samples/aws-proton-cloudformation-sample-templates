## Description

This template is compatible with the [fargate-env](../../environment-templates/fargate-env) template. It creates a scheduled Fargate task that will be initiated off of Amazon EventBridge rules either on a schedule or in response to an EventBridge event. The EventBridge event that you create can run one or more tasks in your cluster at specified times. Your scheduled event can be set to a specific interval (run every N minutes, hours, or days). Otherwise, for more complicated scheduling, you can use a cron expression. The [Schedule Expression](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html) can be specified using the schedule_expression input. For more information, see [Scheduled tasks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/scheduled_tasks.html). The task can be configured to run in a Public or a Private subnet using the subnet_type parameter. Other properties like desired task count, task size (cpu/memory units), and docker image URL can be specified through the service input parameters. 

The template also provisions a CodePipeline based pipeline to pull your application source code before building and deploying it to the Proton service. To use sample application code, please fork the sample code repository [aws-proton-sample-services](https://github.com/aws-samples/aws-proton-sample-services). By default, the template deploys a [python application](https://github.com/aws-samples/aws-proton-sample-services/tree/main/ecs-ping-sns) that sends a random 5-letter string along with the Time to the shared SNS topic, every 5 minutes. 

## Architecture

### Public Subnet
![scheduled-fargate-public-srv](../../images/scheduled-fargate-public-srv.png)

### Private Subnet
![scheduled-fargate-private-srv](../../images/scheduled-fargate-private-srv.png)

## Parameters

### Service Inputs

1. desired_count: The default number of Fargate tasks you want running
2. task_size: The size of the task you want to run
3. subnet_type: Subnet type for your service
4. image: The name/url of the container image
5. schedule_expression: The schedule or rate (frequency) for EventBridge Events

### Pipeline Inputs

1. service_dir: Source directory for the service
2. dockerfile: The location of the Dockerfile to build
3. unit_test_command: The command to run to unit test the application code
4. environment_account_ids: The environment account ids for service instances using cross account environment

## Test
This scheduled service can be tested by deploying the [ecs-ping-sns](https://github.com/aws-samples/aws-proton-sample-services/tree/main/ecs-ping-sns) application that sends a random message to the shared SNS topic, every 5 minutes. We can then deploy a [worker-fargate-svc](../worker-fargate-svc/) that created an SQS, which subscribes to the shared SNS Topic, and runs [ecs-worker](https://github.com/aws-samples/aws-proton-sample-services/tree/main/ecs-worker) application to read the SQS queue and writes the message to CloudWatch logs. Expected data in CloudWatch logs:
```
INFO: The message body: {
  "Type" : "Notification",
  "MessageId" : "605944d3-5636-5e1c-8a77-1e3aaa0dc93b",
  "TopicArn" : "arn:aws:sns:us-east-2:XXXXXXXXXXXX:AWSProton-fargate-env-pro-cloudformation--MPDTYJAPBFYGPYF-ping",
  "Message" : "Hello! Message srptt sent at time Monday, May 02 2022, 15:34:19",
  "Timestamp" : "2022-05-02T15:34:19.387Z",
  "SignatureVersion" : "1",
  "Signature" : "WNHwbHUbTdc7yvJEmY1S9cvGUE0JMuP/PR6GNYe1p7bnyYGcJ7wB3QU3W/5264/VkPN+eMm+FOkW/PZAQr36Tww8TwPMr21xoSZyXfYoyvyk1FecS2i3IMmgRrYBQAi9BbBIhZO+2zBD35640rKAPakvPbNjhV+SNfeF3cPBezlSAgREUd+TovsmI+78h8AIa+dmUaZHFKCFCFmhOo+ovLZGQoLw+H4ow/YofFyzZCr/jNx4iHiI7K15YQ6TPky0S+4xpxMhjJD6vQ2XR75cnZpjKgQ6ip+uTXC4eKYE6mRHW/JBeriwKMv6TaQ3UatiJheyFJ28WRQxAZWeOW1k4g==",
  "SigningCertURL" : "https://sns.us-east-2.amazonaws.com/SimpleNotificationService-7ff5318490ec183fbaddaa2a969abfda.pem",
  "UnsubscribeURL" : "https://sns.us-east-2.amazonaws.com/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:us-east-2:XXXXXXXXXXXX:AWSProton-fargate-env-pro-cloudformation--MPDTYJAPBFYGPYF-ping:445f255f-c616-4852-aa76-b30a40864848"
}

INFO: Deleting message from the queue...

INFO: Received and deleted message(s) from https://sqs.us-east-2.amazonaws.com/XXXXXXXXXXXX/AWSProton-worker-fargate-worker-fargate-c-ServiceEcsProcessingQueue-XWL8D7ptF4mP with message
```

## Security

See [CONTRIBUTING](../../CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](../../LICENSE) file.




