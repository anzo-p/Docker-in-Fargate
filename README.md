# Deploy Docker container in dedicated VPC in AWS Fargate

NOTE: *At least more than a dollar per day* will be charged by AWS for the involved Elastic IP and NAT Gateway (at the time of writing).

Stop that anytime by deleting stack `natgw.yml` while keeping everything else on. Container will respond to requests but can no longer access any further apis or services in the internet (outside your VPC). Should the container for any reason be stopped, then Load Balancer will dissoc from it and Fargate will require internet again to start the container - repull image then run container. Initiate all that by creating stack `natgw.yml` again - Fargate will repull and run, and also Load Balancer will assoc to container again.

*To spare you from all this, look for a free community IAM that handles the 'natgatewaying' for you.*

---
## Why when there is serverless and lambda?

Fargate *is* serverless and Fargate will respond immediately to your requests because it is already online and doesnt require any warm up or prewarming.

## Why the verbose CloudFormation when there are other tools around?

Please use the tool that you find convenient, though bear in mind that they will still likely compose and run CloudFormation code under the hood. And so it might be fun to read the actual syntax of objects, events, and relationships that are happening as your cloud infra gets provisioned. CoudFormation also gives you much control into the process, should you choose to use it.

---

# The process

Have a pullable and runnable docker image available

Set some fields in the yamls:
- public-alb.yml
  - `Mappings.LoadBalancerUrlParts.ListenerProtocol.Value:` and `.TargetGroupProtocol.Value:` to the protocol that you are using
  - `Mappings.LoadBalancerUrlParts.Port.Value:` into the port you wish your Load Balancers to listen to
- private-services.yml
  - `Mappings.ContainerDefinitions.ContainerImagePath.Value:` to the path of your Docker image (format: container-registry/container-name)
  - `Mappings.Endpoints.ContainerPort.Value:` to the port your `Dockerfile EXPOSE`
  - `Mappings.ListenerRuleParts.MainApplicationPath.Value:` to the main application endpoint that your Container image has published
  - `Mappings.TargetGroupParts.HealthCheckPath.Value:` to an actual endpoint in your Container image that will respond 200 OK to the Load Balancer health check.

Install AWS CLI ([link](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)) and Configure it ([link](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html))

Monitor the progress of stack creations in AWS Console > CloudFormation > Stacks and wait for creation to complete before commanding to create the next one.

Create the IAM stack
```
aws cloudformation create-stack --stack-name generic-fargate-iam --template-body file://infra/iam.yml --capabilities CAPABILITY_IAM
# aws cloudformation delete-stack --stack-name generic-fargate-iam
```

Create the VPC stack
```
aws cloudformation create-stack --stack-name generic-fargate-cloud --template-body file://infra/cloud.yml
# aws cloudformation delete-stack --stack-name generic-fargate-cloud
```

Create the NAT Gateway stack - Elastic IP and particularly NAT Gateway will incur a total expense of dollars per day
```
aws cloudformation create-stack --stack-name generic-fargate-natgw --template-body file://infra/natgw.yml
# aws cloudformation delete-stack --stack-name generic-fargate-natgw
```

Create the public Application Load Balancer (ALB) stack
```
aws cloudformation create-stack --stack-name generic-fargate-public-alb --template-body file://infra/public-alb.yml
# aws cloudformation delete-stack --stack-name generic-fargate-public-alb
```

Create the private services stack
```
aws cloudformation create-stack --stack-name generic-fargate-private-services --template-body file://infra/private-services.yml
# aws cloudformation delete-stack --stack-name generic-fargate-private-services
```

Navigate to the URL found in ApiEndpoint at the Output tab of `generic-fargate-private-services` stack and see your service respond to you your content.

## Teardown

Finally teardown entire stack by running respective delete-stack queries in reversed order to the creation of those stacks.