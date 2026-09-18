# Deployment Evidence

Experiment date: September 18, 2026. Region: `us-east-1`.

Milestone 1 and its cleanup are complete. Milestones 2 and 3 have not been
performed yet.

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

Scenario 2 and the healthy redeployment: not performed yet.

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

**The failing curl** (command and output):

```

```

**The log line that told you what was wrong:**

```

```

**What was wrong, and the fix you applied:**

<!-- One or two sentences. Say what you changed and where you changed it. -->

**The healthy curl after the fix:**

```

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

Not performed yet. The evidence above only covers the first healthy deployment.
End Lab has not been clicked as part of this run.
