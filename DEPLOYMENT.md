# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

```
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --region us-east-1 \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
--------------------------------------------------------------------------
|                             DescribeStacks                             |
+------------+-----------------------------------------------------------+
|  InstanceId|  i-0e9e33842df0ce4b8                                      |
|  ServiceUrl|  http://ec2-54-196-173-166.compute-1.amazonaws.com:8080   |
+------------+-----------------------------------------------------------+
```

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-54-196-173-166.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

The template creates a single t3.micro EC2 instance (`ServiceInstance`) running the latest
Amazon Linux 2023 AMI, with the Learner Lab's `LabInstanceProfile` attached (for SSM
sessions) and the `vockey` key pair (for SSH). A security group (`ServiceSecurityGroup`)
allows inbound TCP on the service port (8080) and on port 22 from anywhere, and leaves
outbound open so the instance can install packages and pull the image. On first boot, the
`UserData` script installs and starts Docker, schedules an automatic shutdown after four
hours, and runs the `ghcr.io/cmu-17-214/lab04-service` container with host port 8080
forwarded to container port 8080 and the container's `PORT` environment variable set to
`PortOverride` if given, otherwise `ServicePort`. The stack outputs the service's public
URL and the instance ID.

## 4. Scenario 2 diagnosis

Scenario 2 deploy (`--parameters file://infra/params-scenario2.json`):

```
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-0b802569959109dcf                                     |
|  ServiceUrl|  http://ec2-54-226-44-175.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

**The failing curl** (command and output):

```
$ curl http://ec2-54-226-44-175.compute-1.amazonaws.com:8080/api/health
curl: (28) Failed to connect to ec2-54-226-44-175.compute-1.amazonaws.com port 8080 after 75035 ms: Couldn't connect to server
```

This was run after the container had been up for several minutes, so it is not a warmup
failure. The healthy curl below, from the same machine and network, succeeds, so the
network is not blocking port 8080.

**The log line that told you what was wrong:**

```
$ aws ssm start-session --target i-0b802569959109dcf --region us-east-1

Starting session with SessionId: user5418855=Katie_Zhang-pufg8q5egda4k6i4gp9ua2ak34
sh-5.2$ sudo docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
e6d7139f3921   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   4 minutes ago   Up 4 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service
sh-5.2$ sudo docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

The service listened on port 9090 inside the container (`lab04-service listening on 9090`)
because `params-scenario2.json` set `PortOverride` to 9090, while `ServicePort`, which
drives the security group and the `0.0.0.0:8080->8080/tcp` mapping shown by `docker ps`,
stayed at 8080, so traffic forwarded to container port 8080 found nothing listening.
I fixed it the infrastructure way by deleting the stack and recreating it with
`params-healthy.json` (`PortOverride` empty, so the container listens on `ServicePort`
8080), rather than patching the running container.

**The healthy curl after the fix:**

```
$ aws cloudformation delete-stack --stack-name lab04-service --region us-east-1
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service --region us-east-1
$ aws cloudformation create-stack --stack-name lab04-service \
    --template-body file://infra/template.yaml \
    --parameters file://infra/params-healthy.json \
    --region us-east-1
{
    "StackId": "arn:aws:cloudformation:us-east-1:532843564291:stack/lab04-service/5300a560-b238-11f1-91f6-0affdffb0835",
    "OperationId": "530204f0-b238-11f1-91f6-0affdffb0835"
}
--------------------------------------------------------------------------
|                             DescribeStacks                             |
+------------+-----------------------------------------------------------+
|  InstanceId|  i-02c200e17016ff50a                                      |
|  ServiceUrl|  http://ec2-100-56-234-148.compute-1.amazonaws.com:8080   |
+------------+-----------------------------------------------------------+

$ curl http://ec2-100-56-234-148.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

```
$ aws cloudformation delete-stack --stack-name lab04-service --region us-east-1
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service --region us-east-1
$ aws cloudformation describe-stacks --stack-name lab04-service --region us-east-1

aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```

After teardown, I clicked **End Lab** in the Learner Lab.