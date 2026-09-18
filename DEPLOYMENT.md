# Deployment Evidence

## 1. Deployed URL and instance id

### Healthy deploy (milestone 1)

```
$ aws cloudformation describe-stacks --stack-name lab04-service \
    --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-0cddd77c459a4527d                                    |
|  ServiceUrl|  http://ec2-98-94-19-231.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+
```

- **ServiceUrl:** http://ec2-98-94-19-231.compute-1.amazonaws.com:8080
- **InstanceId:** i-0cddd77c459a4527d

### Healthy redeploy after the fix (milestone 2)

```
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-04779dcd5b331bddb                                     |
|  ServiceUrl|  http://ec2-54-226-240-35.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

- **ServiceUrl:** http://ec2-54-226-240-35.compute-1.amazonaws.com:8080
- **InstanceId:** i-04779dcd5b331bddb

## 2. External health check

Run from my own machine, against the milestone 1 healthy deploy.

```
$ curl http://ec2-98-94-19-231.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}

$ curl http://ec2-98-94-19-231.compute-1.amazonaws.com:8080/api/rooms
[
  {"id":"WEH-5202","name":"Wean 5202","capacity":40},
  {"id":"GHC-4401","name":"Gates 4401","capacity":24},
  {"id":"POS-146","name":"Posner 146","capacity":120},
  {"id":"TEP-2700","name":"Tepper 2700","capacity":16}
]
```

## 3. What the template created

**Compute:** one `t3.micro` EC2 instance running Amazon Linux 2023, with the AMI id
resolved at deploy time from a public SSM parameter rather than hardcoded.

**Network:** a security group on that instance allowing inbound TCP only on 8080 (the
service) and 22 (SSH fallback), with the EC2 default of all outbound traffic, which is
how the instance reaches the internet to install Docker and pull the image.

**What starts the service:** the service itself is a prebuilt image the course
publishes, `ghcr.io/cmu-17-214/lab04-service:latest`. What launches it is the
template's `UserData` script, which EC2 runs once on first boot: it installs Docker,
starts the daemon, then runs that image as a container with `-p 8080:8080` and the
listen port passed in through the `PORT` environment variable. The stack outputs the
instance's public DNS as `ServiceUrl` and its id as `InstanceId`.

## 4. Scenario 2 diagnosis

Deployed with `--parameters file://infra/params-scenario2.json`. New instance, so both
outputs changed:

```
------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-0537cbc6d1d5b6b74                                    |
|  ServiceUrl|  http://ec2-54-164-3-244.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+
```

**The failing curl** (command and output):

```
$ curl http://ec2-54-164-3-244.compute-1.amazonaws.com:8080/api/health
curl: (28) Connection timed out after 20005 milliseconds
```

**The log line that told you what was wrong:**

Opened the instance through SSM (`i-0537cbc6d1d5b6b74`) and ran `docker ps` and
`docker logs lab04-service`. The two outputs side by side are the diagnosis:

```
CONTAINER ID   IMAGE                                     ...  STATUS         PORTS                                       NAMES
1c92ab06e98a   ghcr.io/cmu-17-214/lab04-service:latest   ...  Up 7 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service

lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

A port mismatch *inside* the instance, between the port the app binds and the port
Docker forwards to it. `params-scenario2.json` sets `PortOverride` to `9090`, and the
`UserData` script in `infra/template.yaml` passes that straight through as the
container's `PORT` environment variable, so the app bound **9090**. But the same script hardcodes the host
mapping as `-p ${ServicePort}:${ServicePort}`, that is `-p 8080:8080`, which is what
`0.0.0.0:8080->8080/tcp` reports. So traffic arriving on host 8080 was forwarded to
container port 8080, where nothing was listening, while the app sat unreachable on
9090. 

The fix was to redeploy the stack that leaves
`PortOverride` empty so `EFFECTIVE_PORT` falls back to `ServicePort`, putting the app
and the port mapping both on 8080. Deleted the broken stack and created it again
rather than patching the container over the SSM session, so that the template stays an
accurate description of what is actually deployed.

**The healthy curl after the fix:**

```
$ curl http://ec2-54-226-240-35.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

```

```
