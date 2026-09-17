# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

Healthy deployment (Milestone 1):

InstanceId: `i-0457dda29bf8fce82`

ServiceUrl: `http://ec2-3-94-194-109.compute-1.amazonaws.com:8080`

Scenario 2 broken deployment:

InstanceId: `i-0be09d6c2be074c40`

ServiceUrl: `http://ec2-98-88-255-174.compute-1.amazonaws.com:8080`

Healthy redeployment after the fix:

InstanceId: `i-0c3c2a6013876cc02`

ServiceUrl: `http://ec2-174-129-48-156.compute-1.amazonaws.com:8080`

## 2. External health check

```
$ curl http://ec2-3-94-194-109.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created


The CloudFormation template created a t3.micro EC2 instance running Amazon Linux 2023. Its security group allowed inbound access on port 8080 for the service and port 22 for SSH fallback. At startup, EC2 user data installed and started Docker, then ran the Lab 04 image from GHCR with port 8080 mapped to the host. CloudFormation connected the resources and exposed the instance ID and service URL as stack outputs.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

After the EC2 status checks passed and SSM reported the instance online, the check failed three times; it still failed after the container had been running for 12 minutes.

```
$ curl http://ec2-98-88-255-174.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-98-88-255-174.compute-1.amazonaws.com port 8080 after 53 ms: Couldn't connect to server
```

**The log line that told you what was wrong:**

```
$ sudo docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

SSM `sudo docker ps` showed `0.0.0.0:8080->8080/tcp`, while the application listened on port 9090 because `infra/params-scenario2.json` set `PortOverride` to `9090`. I deleted the broken stack and recreated it with `infra/params-healthy.json`, which leaves `PortOverride` empty so the application listens on port 8080.

**The healthy curl after the fix:**

```
$ curl http://ec2-174-129-48-156.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

After the final healthy check, I deleted `lab04-service` and waited for CloudFormation to finish. A subsequent `DescribeStacks` call confirmed that the stack no longer exists.

```
DELETE_COMPLETE
botocore.exceptions.ClientError: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```
