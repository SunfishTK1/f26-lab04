# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

**Healthy deploy #1 (milestone 1), `params-healthy.json`, 2026-09-18 11:29 EDT:**

```
$ aws cloudformation create-stack --stack-name lab04-service \
    --template-body file://infra/template.yaml \
    --parameters file://infra/params-healthy.json
{
    "StackId": "arn:aws:cloudformation:us-east-1:822690231649:stack/lab04-service/c6770370-b375-11f1-9399-12502256672f"
}
$ aws cloudformation wait stack-create-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-0acdaf210dd98fce0                                     |
|  ServiceUrl|  http://ec2-100-56-235-56.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

**Scenario 2 deploy (milestone 2), `params-scenario2.json`, 2026-09-18 11:36 EDT:**

```
$ aws cloudformation create-stack --stack-name lab04-service \
    --template-body file://infra/template.yaml \
    --parameters file://infra/params-scenario2.json
{
    "StackId": "arn:aws:cloudformation:us-east-1:822690231649:stack/lab04-service/c4d56bf0-b376-11f1-9e51-124a33e4cf99"
}
$ aws cloudformation wait stack-create-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-0ee16d891f3b93cab                                     |
|  ServiceUrl|  http://ec2-100-59-200-28.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

**Healthy deploy #2 (milestone 2 fix), `params-healthy.json`, 2026-09-18 11:42 EDT:**

```
$ aws cloudformation delete-stack --stack-name lab04-service
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
$ aws cloudformation create-stack --stack-name lab04-service \
    --template-body file://infra/template.yaml \
    --parameters file://infra/params-healthy.json
{
    "StackId": "arn:aws:cloudformation:us-east-1:822690231649:stack/lab04-service/80a89cd0-b377-11f1-ac7d-0e73b11ab369"
}
$ aws cloudformation wait stack-create-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-0cd3f9278ed834a37                                    |
|  ServiceUrl|  http://ec2-3-95-238-247.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+
```

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

Run from my laptop about one minute after CREATE_COMPLETE. The first two attempts
(11:31:05 and 11:31:20) returned `curl: (7) ... Couldn't connect to server` because the
instance was still installing Docker and pulling the image; the third attempt succeeded.

```
$ curl http://ec2-100-56-235-56.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

Three or four sentences, your own words. What compute, what network access, and what
glue made the service start.

The template creates exactly two resources. The compute is one `t3.micro` EC2 instance
(`ServiceInstance`, template line 63) running the current Amazon Linux 2023 AMI, which is
resolved at deploy time from a public SSM parameter (line 33); it is attached to the
Learner Lab's `LabInstanceProfile` (line 72), which carries the SSM permissions that let
`aws ssm start-session` open a shell on it, and to the `vockey` key pair (line 74) as an
SSH fallback. Network access comes from one security group (`ServiceSecurityGroup`,
line 42) that allows inbound TCP from anywhere (`0.0.0.0/0`) on the `ServicePort`
parameter (8080) and on port 22, with the EC2 default of all outbound traffic allowed so
the instance can reach the package repos and the container registry. The glue is the
instance's UserData script (lines 80 to 108), which runs once as root on first boot: it
installs and enables Docker, adds `ec2-user` to the docker group, schedules a
`shutdown -h +240` safety stop, computes `EFFECTIVE_PORT` from `PortOverride` (falling
back to `ServicePort`), and runs
`docker run -d --name lab04-service --restart unless-stopped -p 8080:8080 -e PORT=8080 ghcr.io/cmu-17-214/lab04-service:latest`.
The two outputs, `ServiceUrl` (the instance's public DNS name plus `ServicePort`) and
`InstanceId`, are just references to that instance so you can curl it and open a session.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

Polled every 20 seconds from 11:37:43 to 11:40:24 EDT (CREATE_COMPLETE was 11:37:22), so
this is well past the one-to-two-minute warm-up window and the failure persisted on all
nine attempts. The capture below is from about three minutes after CREATE_COMPLETE, and
`docker ps` on the instance (next block) confirms the container had already been up for
two minutes at that point, so this is not a too-early curl.

```
$ curl http://ec2-100-59-200-28.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-100-59-200-28.compute-1.amazonaws.com port 8080 after 115 ms: Couldn't connect to server
```

**The log line that told you what was wrong:**

Read from the instance through SSM. The interactive `aws ssm start-session --target
i-0ee16d891f3b93cab` opened (SessionId `user5456211=Thomas_Kanz-rd4cf2kko2jajkclselsi3x2ra`)
but the non-interactive command variant needs a terminal, so the two commands were run
through SSM Run Command instead, which is the same agent and the same shell on the box.

```
$ aws ssm send-command --instance-ids i-0ee16d891f3b93cab --document-name AWS-RunShellScript \
    --parameters 'commands=["sudo docker ps","echo","sudo docker logs lab04-service"]'
$ aws ssm get-command-invocation --command-id b4727478-efed-4f80-a55f-d3eb0cbd588f \
    --instance-id i-0ee16d891f3b93cab --query StandardOutputContent --output text
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
4e895cf39165   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service

lab04-service listening on 9090
```

The two lines that matter, side by side: `docker ps` says the host forwards
`0.0.0.0:8080->8080/tcp`, and `docker logs` says `lab04-service listening on 9090`.

**What was wrong, and the fix you applied:**

`params-scenario2.json` sets `PortOverride` to `9090` while leaving `ServicePort` at
`8080`. In the UserData script, `EFFECTIVE_PORT` takes `PortOverride` when it is
non-empty (template lines 96 to 99), and that value is passed into the container as the
`PORT` environment variable (line 107, `-e PORT="$EFFECTIVE_PORT"`). The Java service
reads `PORT` and binds it, which is why the log says `listening on 9090`. But the port
mapping on line 106 is `-p ${ServicePort}:${ServicePort}`, i.e. `8080:8080`, and the
security group on line 48 only opens `ServicePort`. So traffic from the internet reaches
the host on 8080, Docker forwards it to port 8080 inside the container, and nothing is
listening there because the process bound 9090 instead. The connection is refused, which
curl reports as `(7) Couldn't connect to server`. The security group and the port
mapping were fine; the wrong port was the one the *process* bound, inside the container,
caused by the `PORT` env var disagreeing with the `-p` mapping.

The fix was infrastructure-level, not a patch on the running box: `aws cloudformation
delete-stack --stack-name lab04-service`, wait for the delete, then `create-stack` again
with `infra/params-healthy.json`, where `PortOverride` is empty so `EFFECTIVE_PORT` falls
back to `ServicePort` and the process binds 8080, matching the mapping and the security
group. That produced a fresh instance (new URL and InstanceId in section 1) whose stack
parameters truthfully describe what is running.

**The healthy curl after the fix:**

CREATE_COMPLETE was 11:42:37 EDT; the first five polls (15 s apart) got "couldn't
connect" during warm-up, and the sixth, at 11:43:54, succeeded.

```
$ curl http://ec2-3-95-238-247.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

Run at 11:44 EDT on 2026-09-18, after capturing the healthy curl in section 4. The
describe command failing is the proof the stack is gone; the two extra queries confirm
no lab stack remains and every instance this lab created is terminated. (The one stack
still listed, `c226398a...`, is the AWS Academy Learner Lab's own environment stack,
not something this lab created.)

```
$ aws cloudformation delete-stack --stack-name lab04-service
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service

An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist

$ aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE CREATE_IN_PROGRESS DELETE_IN_PROGRESS \
    --query "StackSummaries[].StackName" --output text
c226398a5716717l16948221t1w822690231649
$ aws ec2 describe-instances --filters Name=tag:course,Values=17-214 \
    --query "Reservations[].Instances[].[InstanceId,State.Name]" --output text
i-0ee16d891f3b93cab	terminated
i-0acdaf210dd98fce0	terminated
i-0cd3f9278ed834a37	terminated
```

Then **End Lab** was clicked in the Learner Lab page.
