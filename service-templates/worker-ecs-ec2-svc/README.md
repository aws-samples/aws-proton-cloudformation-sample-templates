## worker-ecs-ec2-svc

This template creates an ECS service that allows you to implement asynchronous service-to-service communication with pub/sub architectures (https://aws.amazon.com/pub-sub-messaging/). The microservices in your application can publish events to Amazon SNS topics (https://docs.aws.amazon.com/sns/latest/dg/welcome.html) that can then be consumed by a "Worker Service". A Worker Service is composed of: 1) One or more Amazon SQS queues (https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) to process notifications published to the topics, as well as dead-letter queues (https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) to handle failures, 2) An Amazon ECS service on EC2 that has permission to poll the SQS queues and process the messages asynchronously.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

