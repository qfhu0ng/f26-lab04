# Deployment Evidence

Experiment date: September 18, 2026. Region: `us-east-1`.

All three milestones are complete. All Lab 04 stacks have been deleted;
End Lab has been confirmed in AWS Academy.

## Local warm-up

`mvn -B test` completed successfully:

```text
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

`docker build -t lab04-service .` succeeded. After starting the container with
`docker run -d --rm --name lab04-service -p 8080:8080 lab04-service`, the first
request hit a startup connection reset. A subsequent request succeeded:

```text
$ curl --noproxy '*' --fail --silent --show-error http://localhost:8080/api/health
{"status":"ok"}
$ docker logs lab04-service
lab04-service listening on 8080
$ docker stop lab04-service
lab04-service
```

## 1. Deployed URL and instance id

### Milestone 1: healthy deployment

```sh
aws cloudformation create-stack --stack-name lab04-service \
  --template-body file://infra/template.yaml \
  --parameters file://infra/params-healthy.json \
  --region us-east-1 --no-cli-pager
aws cloudformation wait stack-create-complete --stack-name lab04-service --region us-east-1
aws cloudformation describe-stacks --stack-name lab04-service --region us-east-1 \
  --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table --no-cli-pager
```

The create waiter exited successfully. The outputs were:

```text
--------------------------------------------------------------------------
|                             DescribeStacks                             |
+------------+-----------------------------------------------------------+
|  InstanceId|  i-03ac19e0525c97ca6                                      |
|  ServiceUrl|  http://ec2-54-160-191-206.compute-1.amazonaws.com:8080   |
+------------+-----------------------------------------------------------+
```

### Milestone 2: scenario-2 deployment

Created with `--parameters file://infra/params-scenario2.json`; the creation
waiter completed at 13:32:48 UTC on September 18, 2026.

```text
InstanceId: i-009b3802aecb25460
ServiceUrl: http://ec2-54-227-108-30.compute-1.amazonaws.com:8080
```

### Milestone 2: healthy redeployment

The broken stack was deleted and `stack-delete-complete` returned successfully
before creating a new stack with `infra/params-healthy.json`. The creation waiter
completed at 13:36:33 UTC on September 18, 2026.

```text
InstanceId: i-0355cabcfc59d0deb
ServiceUrl: http://ec2-3-85-226-91.compute-1.amazonaws.com:8080
```

## 2. External health check

Run from my Mac, outside the EC2 instance. Initial requests during startup timed
out; the request at approximately 13:16:41 UTC succeeded (exit code 0):

```text
$ curl --noproxy '*' --connect-timeout 5 --max-time 10 --fail --silent --show-error http://ec2-54-160-191-206.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

The template creates one Amazon Linux 2023 `t3.micro` EC2 instance and attaches a
security group that allows inbound TCP traffic on port 8080 for the service and
port 22 for the SSH fallback. The instance uses the existing Learner Lab
`LabInstanceProfile` for SSM access and the existing `vockey` key pair; the template
does not create these. Its `UserData` script installs and starts Docker, then runs
the course image with host port 8080 forwarded to container port 8080 and the
application's `PORT` set to 8080 for the healthy parameters. The stack outputs the
public service URL and instance ID, and the startup script schedules a shutdown
after four hours as a guard, which does not replace deleting the stack.

## 4. Scenario 2 diagnosis

The instance finished its first-boot script (`sudo cloud-init status --wait`
reported `status: done`). The two external checks at 13:34:24 UTC and
13:34:42 UTC both failed with exit code 28; this was not just an early request
while Docker was being installed.

**The failing curl** (command and output, same error on both attempts):

```text
$ curl --noproxy '*' --connect-timeout 5 --max-time 10 --fail --silent --show-error http://ec2-54-227-108-30.compute-1.amazonaws.com:8080/api/health
curl: (28) Failed to connect to ec2-54-227-108-30.compute-1.amazonaws.com port 8080 after 5005 ms: Timeout was reached
```

**The instance evidence:**

Connected using `aws ssm start-session --target i-009b3802aecb25460 --region us-east-1`.

```text
$ sudo cloud-init status --wait
status: done
$ sudo docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED          STATUS          PORTS                                       NAMES
a330e91c7066   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   23 seconds ago   Up 22 seconds   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service
$ sudo docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix applied:**

The scenario-2 parameter file sets `PortOverride=9090`, so the application listens
on container port 9090, while `ServicePort=8080` keeps the security-group service
rule and Docker mapping at host 8080 to container 8080; the running container's
mapping and the log line above show the mismatch. I fixed the infrastructure by
deleting the broken stack and recreating it with `infra/params-healthy.json`, where
`PortOverride` is empty and the application therefore listens on `ServicePort`
8080, rather than modifying the running container.

**The healthy curl after the fix:**

The request at approximately 13:37:56 UTC succeeded (exit code 0), after two
startup-time retries:

```text
$ curl --noproxy '*' --connect-timeout 5 --max-time 10 --fail --silent --show-error http://ec2-3-85-226-91.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

### Cleanup after Milestone 1

The delete command and delete waiter both completed successfully without output.
The following describe command then exited with code 254 because the stack no
longer existed:

```text
$ aws cloudformation delete-stack --stack-name lab04-service --region us-east-1 --no-cli-pager
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service --region us-east-1
$ aws cloudformation describe-stacks --stack-name lab04-service --region us-east-1 --no-cli-pager
aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```

### Final cleanup after Milestone 2

The final healthy stack was deleted, and `stack-delete-complete` returned
successfully. At 13:39:01 UTC on September 18, 2026, the following check confirmed
that the stack no longer existed (exit code 254):

```text
$ aws cloudformation delete-stack --stack-name lab04-service --region us-east-1 --no-cli-pager
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service --region us-east-1
$ aws cloudformation describe-stacks --stack-name lab04-service --region us-east-1 --no-cli-pager
aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```

After deleting the stack, I selected **End Lab** in AWS Academy and confirmed
**Yes**. The page subsequently reported **AWS Status: Terminated**,
**Lab terminated**, and a session timer of **00:00**.
