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

<!-- filled in during milestone 2 -->

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

**Glue:** a `UserData` script that runs on first boot — it installs Docker, starts the
daemon, and runs the container with `-p 8080:8080`, passing the listen port in through
the `PORT` environment variable. The stack then outputs the instance's public DNS as
`ServiceUrl` and its id as `InstanceId`.

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

```

```
