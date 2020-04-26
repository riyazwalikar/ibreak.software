---
title: "What are these 'reserved' set of security-credentials in AWS?"
date: 2020-04-25
categories:
- cloud
- aws
- penetration testing
- offsec
tags:
- security-credentials
- iam
- aws

thumbnailImagePosition: left
thumbnailImage: /img/the-reserved-set-of-aws-ec2-credentials/1.png
---

A quick blog post to investigate what `instance-identity` security credentials are that can be generated using the metadata instance on every EC2 instance in AWS, even when no role is attached to the instance.

<!--more-->

## Introduction

When we attach a role to an EC2 instance, we can generate temporary AWS credentials by querying the instance meta-data endpoint at `http://169.254.169.254/latest/meta-data/iam/security-credentials/role_name`. This is pretty well known and is used by numerous services and implementations throughout the AWS world. It is also one of the first endpoints that attackers query when a SSRF is discovered on an EC2 hosted web application.

However, in this post, let us try to dig up on the lesser known (and used) endpoint at `http://169.254.169.254/latest/meta-data/identity-credentials/ec2/security-credentials/ec2-instance` that is at best vaguely documented by AWS and does provide, what appears to be security credentials that can be used programmatically.


## AWS Documentation about the endpoint

If you take a look at https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/instancedata-data-categories.html, which describes instance metadata categories on AWS, you will see that the data column mentions `identity-credentials/ec2/security-credentials/ec2-instance` which according to the description is `[Reserved for internal use only] The credentials that AWS uses to identify an instance to the rest of the Amazon EC2 infrastructure.`

This description is as useful as it gets. So I set about trying to figure out what these 'reserved' set of credentials are and what access do they provide if extracted and used.

![](/img/the-reserved-set-of-aws-ec2-credentials/2.png)

## The Setup (if you want to look at this yourself)

All you need is shell access to an EC2 running on AWS. You can launch a free tier instance on AWS and SSH to it. I'm using an Ubuntu 18.04 machine for the rest of this post.

The following curl command will fetch you the credentials that this metadata endpoint generates (IMDSv1).

```bash
curl http://169.254.169.254/latest/meta-data/identity-credentials/ec2/security-credentials/ec2-instance
```

For instances with IMDSv2

```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` \
&& curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/identity-credentials/ec2/security-credentials/ec2-instance
```

![](/img/the-reserved-set-of-aws-ec2-credentials/1.png)

## Analysis of the credentials

From the output, they have the same set of name value pairs as standard role based temporary credentials. The values required to start working with these credentials are `AccessKeyId`, `SecretAccessKey` and `Token`. Make a note of the `Expiration` as well, as the credentials will no longer work after that.

### Using the credentials with the AWS cli

One of the first things I do when I find random credentials during assessments or otherwise is to create an AWS cli profile and figure out who those credentials belong to. These can be done using the following steps

`1.` On a system with aws cli installed, run `aws configure --profile ec2id`

![](/img/the-reserved-set-of-aws-ec2-credentials/3.png)

`2.` Manually edit the `~/.aws/credentials` file in your favorite editor and add the token obtained from the curl command, below the secret as shown below

![](/img/the-reserved-set-of-aws-ec2-credentials/4.png)

You can also use environment variables to do this as described here - https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html

### Running the 'whoami' of AWS IAM

Using the [Security Token Service](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html) of AWS, we can identify who these credentials belong to

```bash
aws sts get-caller-identity --profile ec2id
```

For my test setup, I get

```bash
{
    "UserId": "8953XXXXXXXX:aws:ec2-instance:i-0ab11ba319180ecd4",
    "Account": "8953XXXXXXXX",
    "Arn": "arn:aws:sts::8953XXXXXXXX:assumed-role/aws:ec2-instance/i-0ab11ba319180ecd4"
}
```
which tells us that the role name is `aws:ec2-instance`. This role name contains characters not allowed by AWS when we try to create a role independently of this exercise

![](/img/the-reserved-set-of-aws-ec2-credentials/5.png)

Also, the `UserId` is not in the standard naming convention for any of the AWS Principals as per https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html#principaltable

One key observation is the presence of the `instance-id` of the EC2 instance (which is unique across the AWS EC2 infrastructure) from where these credentials were taken. If we look back at the AWS documentation for this endpoint, the description does say 

`The credentials that AWS uses to identify an instance to the rest of the Amazon EC2 infrastructure` 

So perhaps that is all these credentials are used for.

Let's see what some other tools (and API calls) tell us.

### Evaluating access using enumerate-iam

Let's take a look at what kind of access these credentials have. We can use the `enumerate-iam` tool by Andres Riancho from Github - https://github.com/andresriancho/enumerate-iam. The tool allows us to enumerate the permissions that a given set of credentials have by bruteforcing `get*` and `list*` API calls allowed by the IAM policy.

To setup this tool on your system

```bash
git clone git@github.com:andresriancho/enumerate-iam.git
cd enumerate-iam/
pip install -r requirements.txt
```

To run the tool, pass the `access-key`, `secret-key` and `session-token` parameters as shown below

```bash
./enumerate-iam.py --access-key ASIA5... --secret-key gXjV8... --session-token IQoJb3...
```

Running the tool gave no usefule results after running it overnight, except for (weirdly) the ability to query endpoints for DynamoDB - `dynamodb.describe_endpoints()`. The awscli equivalent for this is

```bash
aws dynamodb describe-endpoints --profile ec2id
```

### Evaluating access using ScoutSuite

Just to ensure coverage, I ran another of my go to tools for AWS security evaluation called ScoutSuite by NCC Group. You can get this from https://github.com/nccgroup/ScoutSuite

Working with this tool is straightforward using docker, instructions for which are available at https://github.com/nccgroup/ScoutSuite/wiki/Docker-Image

Running the docker on the created profile gave slightly different results as seen with the `enumerate-iam` tool, but visibly with lots of `UnauthorizedOperation` errors.

```bash
docker run --rm -t -v /home/user/.aws:/root/.aws:ro -v "$(pwd)/results:/opt/scoutsuite-report" scoutsuite:latest aws --profile ec2id
```

ScoutSuite's HTML report showed that only CloudTrail and IAM was accessible using these credentials.

![](/img/the-reserved-set-of-aws-ec2-credentials/6.png)

However, a awscli check using the following two equivalent commands showed that the results returned in ScoutSuite were false positives

```bash
aws cloudtrail describe-trails --profile ec2id
aws iam get-account-password-policy --profile ec2id
```

![](/img/the-reserved-set-of-aws-ec2-credentials/7.png)

## Final thoughts

Turns out the credentials provide no known or documented form of access to the AWS account or any resources. The only programmatic access the credentials seem to have provided are the ability to list DynamoDB endpoints and list token caller info using the Session Token Service (STS), which provides the `Instance-id` of the instance from where the token was obtained. It is also worth noting that I only tried out `get*`, `list*` and `describe*` operations using 2 tools. Maybe there is a write API that the token can use?

For now, I will go with AWS on this one and consider that these credentials are truly `[Reserved for internal use only]`, unless I find (or anyone else) that the credentials can be used to do something else.

Do let me know if you do find any other API calls that the credentials can successfully use and I will update the post! Till I write again, Happy Hacking!

## References

- https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/instancedata-data-categories.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html#principaltable

---