docker-ecs-deployment
=====================

This repository contains a Dockerfile of ecs-deployment for Docker's automated build published to the public [Docker Hub Registry](https://registry.hub.docker.com/).

ecs-deployment
--------------

Script for deploying new task definitions to services in Amazon ECS. New definition and service name is provided via an SQS message queue.

Used to perform blue/green deployments of new docker images to production services running in ECS.

The intention is for this container to be run as an ECS task whenever a "watcher" script detects a new message is in the SQS queue.

_Credit: The main body (and workflow) of this script is original, however it was inspired by [ecs-deploy](https://github.com/silinternational/ecs-deploy) and the code responsible for registering the new definiton, updating the service and then reporting success has been copied (as permitted by the MIT License). Kudos and thanks to [SIL International](https://github.com/silinternational/)._

### Requirements

1. An AWS CLI config dir (.aws) with credentials capable of receiving/deleting from the sqs queue and the ecs functions for listing/describing tasks and services, registering task definitions and updating services. The minimum required permissions would be covered by the following IAM policy:

   ```
	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Sid": "<SID>",
	      "Effect": "Allow",
	      "Action": [
	        "sqs:DeleteMessage",
	        "sqs:ReceiveMessage"
	      ],
	      "Resource": [
	        "arn:aws:sqs:<REGION>:<ACCOUNT>:<QUEUE_NAME>"
	      ]
	    },
	    {
	      "Sid": "<SID>",
	      "Effect": "Allow",
	      "Action": [
	        "ecs:DescribeServices",
	        "ecs:DescribeTasks",
	        "ecs:ListTasks",
	        "ecs:RegisterTaskDefinition",
	        "ecs:UpdateService"
	      ],
	      "Resource": [
	        "*"
	      ]
	    }
	  ]
	}
   ```
   
2. A volume mount of that .aws dir to ```/root/.aws``` in the container (see included `docker-compose.yml` or `aws-task-definiton.json` for example)

3. The following ENV variables 

   `SQS_URL=<YOUR_SQS_URL>`
   
   `ECS_CLUSTER=<TARGET_ECS_CLUSTER>` (Optional: defaults to "default")
   
   `ECS_TIMEOUT=<MAX_WAIT_FOR_SUCCESS>` (Optional: defaults to 300 seconds)
   
4. The SQS messages in the queue providing the new task definitions must have:
 - the complete **Task Definition JSON** as the **Message Body** (SQS will escape the JSON payload as required)
 - the target **ECS Service name** set as a **Message Attribute** named **"service"**
   Example (aws-cli): `aws sqs send-message --queue-url "<SQS_URL>"  --message-body "<NEW_TASK_DEF>" --message-attributes '{"service" : { "DataType":"String", "StringValue":"<SERVICE_NAME>"}}'`
