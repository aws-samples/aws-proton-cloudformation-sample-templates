## fargate-env

This template creates a VPC with two public and private subnets across two availability zones.
The VPC includes an Internet Gateway and a managed NAT Gateway in each public subnet as well as VPC Route Tables that allow for communication between the public and private subnets. 

It also deployes a ECS Fargate cluster to host your containers without managing EC2 instances. A private namespace based on DNS, which is visible only inside the VPC, is created for Service Discovery.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

