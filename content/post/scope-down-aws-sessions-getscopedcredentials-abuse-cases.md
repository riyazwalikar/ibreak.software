---
title: "Poking at AWS STS GetScopedCredentials: Use, Abuse and Edge Cases"
date: 2026-11-24T12:00:00+05:30
categories:
- aws
- cloud-security
- agentic-ai

tags:
- sts
- getscopedcredentials
- iam
- session-policy
- imds
- cloudtrail
- incident-response
- ai-agents

thumbnailImagePosition: left
thumbnailImage: /img/scope-down-aws-sessions-getscopedcredentials-abuse-cases/thumbnail.png
---

A blog post about testing the new AWS STS `GetScopedCredentials` API from an attacker's point of view. I wanted to know whether a scope policy can grant more access than the caller if explicitly specified, whether the API works with EC2 instance credentials from IMDS, how far it can be chained, what happens to scoped credentials when the parent access key or credential is deleted/expired, and what shows up in CloudTrail. I used my personal AWS account for this testing using `us-west-2` and the latest AWS CLI at the time of writing.

<!--more-->
---

You can skip to the section that interests you

* [Introduction](#introduction)
* [Use cases](#use-cases)
* [Test setup](#test-setup)
* [Observations](#observations)
  * [Long-term credentials are rejected](#long-term-credentials-are-rejected)
  * [The caller needs sts:GetScopedCredentials](#the-caller-needs-stsgetscopedcredentials)
  * [A scope policy cannot grant more than the caller has](#a-scope-policy-cannot-grant-more-than-the-caller-has)
  * [Scoped credentials cannot outlive the parent](#scoped-credentials-cannot-outlive-the-parent)
  * [IMDS credentials work directly](#imds-credentials-work-directly)
  * [EC2 credential context keys survive the scope down](#ec2-credential-context-keys-survive-the-scope-down)
  * [Chaining is possible but stops at the third generation](#chaining-is-possible-but-stops-at-the-third-generation)
  * [Scoped credentials inherit the parent's aws:TokenIssueTime](#scoped-credentials-inherit-the-parents-awstokenissuetime)
  * [Console federation honours the scope](#console-federation-honours-the-scope)
* [Abuse cases](#abuse-cases)
  * [Escaping the scope with sts:AssumeRole](#escaping-the-scope-with-stsassumerole)
  * [Deleting the access key does not kill the session or its children](#deleting-the-access-key-does-not-kill-the-session-or-its-children)
  * [Scoped activity looks like the parent in CloudTrail](#scoped-activity-looks-like-the-parent-in-cloudtrail)
  * [Control characters in the session description](#control-characters-in-the-session-description)
  * [Principal tags get copied into every CloudTrail event](#principal-tags-get-copied-into-every-cloudtrail-event)
  * [Credentials and policies on the command line](#credentials-and-policies-on-the-command-line)
* [Summary table](#summary-table)
* [Recommendations](#recommendations)
* [What I did not test](#what-i-did-not-test)
* [Final thoughts](#final-thoughts)
* [References](#references)

## Introduction

AWS shipped a new [STS API called `GetScopedCredentials` a few days ago](https://aws.amazon.com/blogs/security/scope-down-permissions-of-existing-sessions-using-aws-sts-getscopedcredentials/). You call it with temporary credentials you already have, pass a policy, a shorter duration or both, and get back a new set of temporary credentials with the same identity but fewer permissions. The CLI puts it this way:

> Returns a set of temporary security credentials that are scoped down from the caller's existing temporary security credentials. GetScopedCredentials preserves the caller's identity while narrowing the permissions and optionally shortening the session lifetime.

Until now, if you held a role session and wanted to hand a narrower copy of it to something else, you had to call `AssumeRole` again with a session policy (an extra policy passed at call time that can only remove permissions). That needs a role that trusts you, a new role session name, and a new identity in CloudTrail. With an IAM user you had `GetFederationToken`, which only works with long-term keys. `GetScopedCredentials` lets any temporary session reduce its scope in place, with no role trust policy involved and the identity remains unchanged.

The obvious consumers are AI agents (as the announcement from AWS also highlights). An agent that runs with your session, or with the instance role of the box it lives on, often spawns sub-agents, tools and MCP servers that need a smaller set of permissions than the parent. The first call I made used `--scoped-session-description "sub-agent-s3-reader"` because that is the kind of workload I expect this API to end up in, and I wanted to poke at it to understand its working and edges.

The API takes four parameters:

| Parameter | What it does |
|---|---|
| `--policy` | Scope down policy, max 2,048 characters. The new credentials can only do what both **this** policy and the caller's own permissions allow |
| `--duration-seconds` | 1 to 129,600 seconds (36 hours). Defaults to the caller's remaining lifetime |
| `--minimum-session-token-size` | Pads the session token up to this many bytes, max 4,096 |
| `--scoped-session-description` | Free text, up to 256 characters, logged to CloudTrail |

The API is not available on the STS global endpoint (`sts.amazonaws.com`), so you need a regional one like `sts.us-west-2.amazonaws.com`.

I went in with two main questions:

1. Can the scope policy be outside the permissions of the IAM entity? For example, if a user can only call `ec2:DescribeInstances`, can a scope policy give the resulting credentials S3 access?
2. Can an agent running on an EC2 instance use the IMDS role credentials to generate scoped credentials, and does the first scenario apply there?

Then I went looking for anything else that could turn into a design problem or a vulnerability because of how this gets used.

## Use cases

These are the places I expect `GetScopedCredentials` to show up, and so the places worth testing when you come across it during a pentest:

- Agent to sub-agent delegation, where a coding or ops agent with a broad session generates a scoped S3 read-only session to a sub-agent that summarises logs.
- Tools and MCP servers, where a harness mints a short-lived scoped credential per tool call, so a compromised or prompt-injected tool only has the permissions the scope defines.
- Third-party scripts and CI steps, where a vendor script or a build step runs with a 15 minute credential that can only touch one bucket.
- Shortening a long session, for example turning a 12 hour session into a 5 minute one before passing it to something less trusted. `--duration-seconds` on its own is a valid request without the `--policy` option.
- Labelling delegated tasks. The description is written to CloudTrail, so each task can carry a label like `sub-agent-s3-reader`.

## Test setup

I used my personal AWS account to create resources, test stuff and removed them at the end:

- `gsc-test-lowpriv`, an IAM user with an inline policy allowing only `ec2:DescribeInstances` (and later `sts:GetScopedCredentials`)
- `gsc-test-ec2role`, an EC2 instance role with probe permissions (more on that below) and `AmazonSSMManagedInstanceCore`, attached to a `t4g.micro` Amazon Linux 2023 instance with IMDSv2 enabled
- `gsc-test-consolerole`, a role with `ReadOnlyAccess` that the admin user could assume

> General paranoia on my end - Account IDs, access key IDs and role/user unique IDs in the output below are masked. AWS access key IDs encode the account number, so I masked those as well. Secrets and session tokens are truncated.

## Observations

### Long-term credentials are rejected

Calling the API with an IAM user's `AKIA` keys fails straight away:

```
$ aws sts get-scoped-credentials \
  --policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:GetObject","Resource":"arn:aws:s3:::amzn-s3-demo-bucket/*"}]}' \
  --duration-seconds 900 \
  --scoped-session-description "sub-agent-s3-reader" \
  --region us-west-2

aws: [ERROR]: An error occurred (AccessDenied) when calling the GetScopedCredentials operation: Cannot call GetScopedCredentials with long-term credentials. Use GetSessionToken to obtain temporary security credentials first and then call GetScopedCredentials with those credentials.
```

Turns out, long term credentials cannot be used to create scoped session tokens. Getting a session token first and repeating the call works:

```
$ export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s" \
  $(aws sts get-session-token \
  --duration-seconds 3600 \
  --query "Credentials.[AccessKeyId,SecretAccessKey,SessionToken]" \
  --output text))

$ aws sts get-scoped-credentials \
  --policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:GetObject","Resource":"arn:aws:s3:::amzn-s3-demo-bucket/*"}]}' \
  --duration-seconds 900 \
  --scoped-session-description "sub-agent-s3-reader" \
  --region us-west-2
{
    "Credentials": {
        "AccessKeyId": "ASIAXXXXXXXXHBJMDPJB",
        "SecretAccessKey": "VsrI9rFnbFOKro-TRUNCATED",
        "SessionToken": "IQoJb3JpZ2luX2Vj-TRUNCATED",
        "Expiration": "2026-11-23T16:47:03+00:00"
    },
    "SessionInfo": {
        "ScopedSessionId": "AIDAXXXXXXXXHAFKU3GUNG",
        "Arn": "arn:aws:iam::123456789012:user/iamadmin"
    },
    "SessionTokenUtilization": 16,
    "SessionTokenSize": 676,
    "ScopedPoliciesCount": 1
}
```

The `ScopedSessionId` for an IAM user is the user's unique ID (`AIDA...`) and the ARN is the user ARN, so the identity stays the same as the caller's.

### The caller needs sts:GetScopedCredentials

My low privilege user, with only `ec2:DescribeInstances`, could get a session token (`aws sts get-session-token` needs no extra permission) but could not scope it:

```
aws: [ERROR]: An error occurred (AccessDenied) when calling the GetScopedCredentials operation: User: arn:aws:iam::123456789012:user/gsc-test-lowpriv is not authorized to perform: sts:GetScopedCredentials on resource: arn:aws:sts::123456789012:self because no identity-based policy allows the sts:GetScopedCredentials action
```

So this is an IAM action, `sts:GetScopedCredentials`, evaluated against a resource called `arn:aws:sts::<account-id>:self`. Admins with `*:*` already have it. Any role with `sts:*` already has it too, which covers a lot of real-world roles.

### A scope policy cannot grant more than the caller has

This answers the first question. I added `sts:GetScopedCredentials` to the low privilege user and asked for a scope that allowed `s3:*`, `iam:*`, `sts:*` and `ec2:*`:

```
$ aws sts get-scoped-credentials \
  --policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:*","iam:*","sts:*","ec2:*"],"Resource":"*"}]}' \
  --scoped-session-description q1-escalate
{
  "Credentials": {
    "AccessKeyId": "ASIAXXXXXXXXEBH67LRB",
    "Expiration": "2026-11-23T19:26:43+00:00"
  },
  "SessionInfo": {
    "ScopedSessionId": "AIDAXXXXXXXXJMTDXWTRX",
    "Arn": "arn:aws:iam::123456789012:user/gsc-test-lowpriv"
  },
  "SessionTokenUtilization": 16,
  "SessionTokenSize": 676,
  "ScopedPoliciesCount": 1
}
```

STS issued the credentials without complaint, which was expected but using them showed the actual permission access:

```
s3 ls
aws: [ERROR]: An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::123456789012:user/gsc-test-lowpriv is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
```

```
iam list-users
aws: [ERROR]: An error occurred (InvalidClientTokenId) when calling the ListUsers operation: The security token included in the request is invalid
```

```
ec2 describe-instances (allowed both)
0
```

```
ec2 describe-vpcs (scope yes, identity no)
aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the DescribeVpcs operation: You are not authorized to perform this operation. User: arn:aws:iam::123456789012:user/gsc-test-lowpriv is not authorized to perform: ec2:DescribeVpcs because no identity-based policy allows the ec2:DescribeVpcs action
```

The scoped credentials can only do what both the user's own policy and the scope policy allow. Session policies on `AssumeRole` work the same way. The `InvalidClientTokenId` on IAM is inherited from the parent: credentials from `GetSessionToken` without MFA cannot call the IAM API, and the scoped child keeps that restriction.

The error wording tells you which layer denied the call. When the identity policy is the reason, the error says "no identity-based policy allows". When the scope is the reason, it says "no session policy allows". Useful to know when you are testing someone else's scoped credentials and want to work out what the parent identity can do.

### Scoped credentials cannot outlive the parent

The help text says `--duration-seconds` goes up to 36 hours, which made me wonder whether a 15 minute session of the parent token could be stretched. It cannot:

```
aws: [ERROR]: An error occurred (InsufficientSessionLifetime) when calling the GetScopedCredentials operation: The requested DurationSeconds exceeds the remaining lifetime of the calling session. Remaining session lifetime is 851 seconds.
```

Leaving the duration out gives the child the parent's expiry.

### IMDS credentials work directly

This answered my second question. The Amazon Linux 2023 AMI I used had an older AWS CLI (`aws-cli/2.38.0`).

If you are lazy like me and don't want to update the CLI on the instance, `curl` can sign SigV4 requests on its own (`--aws-sigv4`), so I pulled the role credentials from IMDSv2 and called the STS query API directly from the instance:

```bash
T=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 300')
R=$(curl -s -H "X-aws-ec2-metadata-token: $T" http://169.254.169.254/latest/meta-data/iam/security-credentials/)
C=$(curl -s -H "X-aws-ec2-metadata-token: $T" http://169.254.169.254/latest/meta-data/iam/security-credentials/$R)
# AK, SK, TK parsed out of $C

curl -s --aws-sigv4 "aws:amz:us-west-2:sts" --user "$AK:$SK" -H "x-amz-security-token: $TK" \
  --data-urlencode Action=GetScopedCredentials --data-urlencode Version=2011-06-15 \
  --data-urlencode 'Policy={"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}' \
  --data-urlencode ScopedSessionDescription=imds-agent \
  https://sts.us-west-2.amazonaws.com/
```

No `GetSessionToken` step is needed because instance role credentials are already temporary:

```xml
<GetScopedCredentialsResponse xmlns="https://sts.amazonaws.com/doc/2011-06-15/">
  <GetScopedCredentialsResult>
    <SessionInfo>
      <ScopedSessionId>AROAXXXXXXXXAK53FCKNW:i-052303c1f8882247e</ScopedSessionId>
      <Arn>arn:aws:sts::123456789012:assumed-role/gsc-test-ec2role/i-052303c1f8882247e</Arn>
    </SessionInfo>
    <ScopedPoliciesCount>1</ScopedPoliciesCount>
    <Credentials>
      <AccessKeyId>ASIAXXXXXXXXM6NFLQW2</AccessKeyId>
      <SecretAccessKey>REDACTED</SecretAccessKey>
      <SessionToken>REDACTED</SessionToken>
      <Expiration>2026-11-24T01:49:17Z</Expiration>
    </Credentials>
    <SessionTokenSize>1316</SessionTokenSize>
    <SessionTokenUtilization>32</SessionTokenUtilization>
  </GetScopedCredentialsResult>
  ...
</GetScopedCredentialsResponse>
```

The role had no S3 permissions, and the scoped credentials with `"Action":"*"` still could not call `s3 ls`. Instance roles get the same treatment as users: the scope can only remove permissions.

The instance role session token is about twice the size of the IAM user one (1316 bytes vs 676), and `SessionTokenUtilization` went from 16 to 32. I assume that is the EC2 context carried inside the token.

### EC2 credential context keys survive the scope down

Instance role credentials carry extra condition keys: `ec2:SourceInstanceARN`, `aws:ec2InstanceSourceVPC` and `aws:ec2InstanceSourcePrivateIPv4`. Data perimeter policies use these to stop instance credentials being used outside the VPC they were issued to, and GuardDuty uses the instance identity for its credential exfiltration findings. If a scoped child dropped these keys, a deny statement that relies on them would stop matching and the scoped credentials could be used from anywhere.

To check, I gave the role probe permissions where each action only works if a specific key is present:

```json
{"Effect":"Allow","Action":"ec2:DescribeVpcs","Resource":"*","Condition":{"Null":{"ec2:SourceInstanceARN":"false"}}},
{"Effect":"Allow","Action":"ec2:DescribeSubnets","Resource":"*","Condition":{"Null":{"aws:ec2InstanceSourceVPC":"false"}}},
{"Effect":"Allow","Action":"ec2:DescribeSecurityGroups","Resource":"*","Condition":{"Null":{"aws:ec2InstanceSourcePrivateIPv4":"false"}}}
```

On the instance, both sets of credentials passed every check:

```
== IMDS creds probes (on-instance)
  ec2 describe-instances       1
  ec2 describe-vpcs            1
  ec2 describe-subnets         4
  ec2 describe-security-groups 3

== scoped creds probes (on-instance)
  ec2 describe-instances       1
  ec2 describe-vpcs            1
  ec2 describe-subnets         4
  ec2 describe-security-groups 3
```

I then copied the credentials to my laptop and ran the same probes. This is the output for the scoped credentials:

```
== ec2-scoped from laptop
  sts get-caller-identity        "arn:aws:sts::123456789012:assumed-role/gsc-test-ec2role/i-052303c1f8882247e"
  ec2 describe-instances         1
  ec2 describe-vpcs              1
  ec2 describe-subnets           4
  ec2 describe-security-groups   3
  s3api list-buckets             aws: [ERROR]: An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:sts::123456789012:assumed-role/gsc-test-ec2role/i-052303c1f8882247e i
```

The keys are part of the token and travel with the scoped child, so perimeter conditions written against them still apply. CloudTrail also kept the instance binding on calls made with the scoped key:

```json
"inScopeOf": {
  "issuerType": "AWS::EC2::Instance",
  "credentialsIssuedTo": "arn:aws:ec2:us-west-2:123456789012:instance/i-052303c1f8882247e"
}
```

### Chaining is possible but stops at the third generation

Scoped credentials can call `GetScopedCredentials` again, but only if their own scope allows `sts:GetScopedCredentials`. Without it, even a duration-only request fails, and the error now blames the session policy:

```
-- a1 tries to re-scope with *
aws: [ERROR]: An error occurred (AccessDenied) when calling the GetScopedCredentials operation: User: arn:aws:iam::123456789012:user/iamadmin is not authorized to perform: sts:GetScopedCredentials on resource: arn:aws:sts::123456789012:self because no session policy allows the sts:GetScopedCredentials action
-- a1 tries duration-only (no policy)
aws: [ERROR]: An error occurred (AccessDenied) when calling the GetScopedCredentials operation: User: arn:aws:iam::123456789012:user/iamadmin is not authorized to perform: sts:GetScopedCredentials on resource: arn:aws:sts::123456789012:self because no session policy allows the sts:GetScopedCredentials action
```

With the permission included, a child can re-scope itself with `"Action":"*"`, and every earlier scope still applies. An admin session scoped to `ec2:DescribeInstances` and `sts:GetScopedCredentials`, then re-scoped to `*`, still could not list buckets. In the output below, `c` is `ScopedPoliciesCount`, `u` is `SessionTokenUtilization` and `s` is `SessionTokenSize`:

```
{"c":2,"u":16,"s":676}
-- a2b s3 ls (should be denied)
aws: [ERROR]: An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::123456789012:user/iamadmin is not authorized to perform: s3:ListAllMyBuckets because no session policy allows the s3:ListAllMyBuckets action
```

A loop re-scoping the same session hit a hard limit at the fourth call:

```
{"depth":1,"c":1,"u":16,"s":676}
{"depth":2,"c":2,"u":16,"s":676}
{"depth":3,"c":3,"u":16,"s":676}
depth 4:
aws: [ERROR]: An error occurred (ValidationError) when calling the GetScopedCredentials operation: Credentials have reached the maximum scope down limit and cannot be scoped down further.
```

`ScopedPoliciesCount` counts the stacked scopes, and the maximum is 3. Scoped credentials also cannot get back to anything wider through the other STS paths:

```
-- a1 get-session-token
aws: [ERROR]: An error occurred (AccessDenied) when calling the GetSessionToken operation: Cannot call GetSessionToken with session credentials

== GetFederationToken w/ session creds
aws: [ERROR]: An error occurred (AccessDenied) when calling the GetFederationToken operation: Cannot call GetFederationToken with session credentials
```

### Scoped credentials inherit the parent's aws:TokenIssueTime

The standard way to revoke a leaked session is a deny on `aws:TokenIssueTime` older than "now", which is what the console's "Revoke active sessions" button writes for roles. If scoped children got a fresh issue time, a child minted after the revocation timestamp would not match that deny.

I minted a scoped child from the low privilege user's session, picked a timestamp between the parent's issue time and the child's (`TMID` in the output below), and attached this to the user:

```json
{"Effect":"Deny","Action":"ec2:DescribeInstances","Resource":"*","Condition":{"DateLessThan":{"aws:TokenIssueTime":"2026-11-23T19:16:15Z"}}}
```

The parent, the child minted after the timestamp and an older child were all denied:

```
-- parent:
aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation. User: arn:aws:iam::123456789012:user/gsc-test-lowpriv is not authorized to perform: ec2:DescribeInstances with an explicit deny in an identity-based policy

-- scoped (minted after TMID):
aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: ... with an explicit deny in an identity-based policy

-- q1-scoped (minted before TMID):
aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: ... with an explicit deny in an identity-based policy
```

Scoped credentials carry the parent's issue time, so existing session revocation catches them. CloudTrail agrees: calls made with a scoped key show the parent's `creationDate`.

### Console federation honours the scope

Role session credentials can be exchanged for an AWS console sign-in URL through `https://signin.aws.amazon.com/federation`. If the console session was built from the role and ignored the scope, that would be a straight escalation.

I assumed `gsc-test-consolerole` (`ReadOnlyAccess`), scoped it down to `ec2:DescribeInstances`, and exchanged the scoped credentials for a sign-in token. The federation endpoint returned a token, and the sign-in URL logged me into the console as `gsc-test-consolerole/gsc-console`. The S3 console then showed:

```
General purpose buckets (0)
You don't have permissions to list buckets
```

The account has 66 buckets and the role can read all of them. The console session kept the scope, so this path holds up.

## Abuse cases

None of the findings above is a vulnerability in the API itself. In every path I tried, the scoped credentials could not do anything the parent could not. The problems below come from how the API behaves together with the rest of IAM, and from how it is likely to be used.

### Escaping the scope with sts:AssumeRole

A scoped session can call `sts:AssumeRole` if the scope allows it. The new role session gets the target role's permissions and none of the scope:

```
== scope=["ec2:DescribeInstances"] -> AssumeRole:
aws: [ERROR]: An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::123456789012:user/iamadmin is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::123456789012:role

== scope=["ec2:DescribeInstances","sts:AssumeRole"] -> AssumeRole:
  assumed; s3 buckets visible: 66
```

This is how session policies have always worked. The difference now is that scopes will often be written at runtime by agent harnesses. A harness that scopes a sub-agent with something like `"Action":["s3:*","sts:*"]` will give that sub-agent every role the parent can assume. The same applies to anything that creates a new principal or runs code as another one: `iam:PassRole` with `lambda:CreateFunction` or `ec2:RunInstances`, `ssm:SendCommand` to an instance with a bigger role, `iam:CreateAccessKey` on another user, `codebuild:StartBuild` with a project override.

Anything a scope allows that leads to another principal leaves the scope behind, so review scopes the same way you would review a session policy.

### Deleting the access key does not kill the session or its children

During incident response for a leaked IAM user key, the first move is usually to deactivate or delete the key. I took a fresh `GetSessionToken` session from the low privilege user, minted a scoped child from it, deactivated the access key, and tried both:

```
-- lp2-sess (key inactive):
1
-- lp2-scoped (key inactive):
1
-- mint:
"ASIAXXXXXXXXKT622JI3"
```

Both kept working, and the parent minted a brand new scoped access key ID after the key was deactivated. I then deleted the key outright:

```
KEY-DELETED
-- lp2-sess (key deleted):
1
-- lp2-scoped (key deleted):
1
```

Both still worked. The `GetSessionToken` part is existing STS behaviour (session tokens live until they expire), but `GetScopedCredentials` adds something to it. An attacker who grabbed a session token before the key was deleted can keep producing fresh `ASIA` key IDs, up to the parent's expiry, which for `GetSessionToken` for an IAM user can be 36 hours. Detections or blocklists pointing to the access key ID that was seen first will not match the new ones.

The `aws:TokenIssueTime` deny from the earlier section, or a deny-all policy on the user, stops the parent and every scoped child.

### Scoped activity looks like the parent in CloudTrail

Here is the `userIdentity` of a `ListBuckets` call made with scoped credentials minted from the EC2 instance role:

```json
{
  "eventName": "ListBuckets",
  "sourceIPAddress": "54.188.81.117",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAXXXXXXXXAK53FCKNW:i-052303c1f8882247e",
    "arn": "arn:aws:sts::123456789012:assumed-role/gsc-test-ec2role/i-052303c1f8882247e",
    "accountId": "123456789012",
    "accessKeyId": "ASIAXXXXXXXXM6NFLQW2",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAXXXXXXXXAK53FCKNW",
        "arn": "arn:aws:iam::123456789012:role/gsc-test-ec2role",
        "accountId": "123456789012",
        "userName": "gsc-test-ec2role"
      },
      "attributes": {
        "creationDate": "2026-11-23T19:14:17Z",
        "mfaAuthenticated": "false"
      },
      "ec2RoleDelivery": "2.0"
    },
    "inScopeOf": {
      "issuerType": "AWS::EC2::Instance",
      "credentialsIssuedTo": "arn:aws:ec2:us-west-2:123456789012:instance/i-052303c1f8882247e"
    }
  }
}
```

The event does not record that this was a scoped session, which scope applied, or what the description was. The ARN, the principal ID and the `creationDate` match the parent session exactly. The only field that differs is `accessKeyId`. To tell what the sub-agent did apart from what the parent agent (or the human) did, you have to find the `GetScopedCredentials` event and do a join on `responseElements.credentials.accessKeyId`.

From an attacker's point of view this is useful. Actions taken with a scoped key blend into the parent's activity for anyone reading by ARN, and every mint produces a new key ID that has not been seen before.

### Control characters in the session description

The allowed pattern for `--scoped-session-description` is `[\u0009\u000A\u000D -~¡-ÿ]*`, which includes tab, line feed and carriage return. I sent a description with a newline, a fake CloudTrail-looking JSON blob, a carriage return, a tab and `ÿ`:

```bash
D=$(printf 'agent-1\n{"eventName":"DeleteTrail","userIdentity":{"arn":"arn:aws:iam::111111111111:root"}}\r\tÿ')
aws sts get-scoped-credentials --policy "$P" --scoped-session-description "$D"
```

It was accepted and stored verbatim:

```json
"scopedSessionDescription":"agent-1\n{\"eventName\":\"DeleteTrail\",\"userIdentity\":{\"arn\":\"arn:aws:iam::111111111111:root\"}}\r\tÿ"
```

CloudTrail JSON-escapes it, so the event itself is fine. The risk sits downstream, in any SIEM parser, log shipper, Slack alert or dashboard that renders the description as raw text and splits on newlines. In an agent harness the description is likely to be built from the task description generated by the model, and a prompt injection in anything the model read can influence that text resulting in second order prompt injection/XSS type vulnerabilities. Treat it as attacker controlled when you display it.

### Principal tags get copied into every CloudTrail event

The `GetScopedCredentials` event includes an `incomingNonTransitiveTags` field in `requestParameters`. For the EC2 role it contained the role's tag. For my admin user it contained this:

```json
{"incomingNonTransitiveTags":{"AKIAXXXXXXXXMHLGDFKT":"created-as-admin-iam-user"}, ...}
```

IAM stores access key descriptions as user tags pointing to the access key ID, so every principal tag, including access key descriptions, ends up in every `GetScopedCredentials` event. Anyone who can read the trail (a log bucket, a SIEM, a security tooling vendor) can now read principal tags without any IAM read permission. It is a small leak, but tags often hold owner emails, cost centres and notes like the one above.

The same event logs the full scope policy, which tells a reader exactly what the sub-agent was allowed to do. It also logs the full `sessionToken` of the new credentials in `responseElements`. The secret access key is not logged, and `AssumeRole` events log the session token the same way, so on its own this is not usable.

### Credentials and policies on the command line

When this API is called from scripts or agent harnesses through the CLI, the obvious leaks apply:

- `--policy` and `--scoped-session-description` end up in `ps` output, shell history and process accounting on the host. The policy tells the reader what the scoped credentials can reach.
- The response prints the secret access key and session token to stdout. Many agent frameworks log tool stdout by default, and that output often goes back into the model's context, into transcripts and into observability tools.
- `--minimum-session-token-size` pads tokens up to 4,096 bytes on request. I confirmed the padding (`{"u":100,"s":4096,"len":4096}`) but did not test what it breaks. Anything with a header size limit, or that stores tokens in a fixed-width field, is worth a look.

## Summary table

| Test | Result |
|---|---|
| Calling with an IAM user's long-term keys | Rejected. You need to call `GetSessionToken` first and use those temporary credentials |
| Permission needed to call the API | `sts:GetScopedCredentials` on `arn:aws:sts::<account>:self` |
| Scope policy that allows more than the user has | STS issues the credentials, but they can only do what both the user's policy and the scope allow |
| Duration longer than the parent's remaining lifetime | Rejected with `InsufficientSessionLifetime` |
| EC2 instance role credentials from IMDS | Work directly, no `GetSessionToken` needed |
| Scope of `*` on a role with no S3 permissions | S3 calls are still denied |
| EC2 condition keys on the scoped credentials | Kept, even when the credentials are used from outside the instance |
| Re-scoping when the scope does not allow `sts:GetScopedCredentials` | Denied, even for a duration-only request |
| Re-scoping with `*` | Earlier scopes still apply, so nothing is gained |
| Maximum chain depth | 3 scopes stacked on one session |
| `GetSessionToken` or `GetFederationToken` with scoped credentials | Denied |
| `aws:TokenIssueTime` on scoped credentials | Same as the parent, so revoking the parent's sessions also revokes the children |
| Console sign-in with scoped credentials | Works, and the console session keeps the scope |
| Scope that allows `sts:AssumeRole` | The assumed role session gets the target role's full permissions, with no scope applied |
| Parent access key deactivated or deleted | The parent session and its scoped children keep working, and new children can still be created |
| CloudTrail entries for scoped activity | Identical to the parent's, apart from the `accessKeyId` |
| Control characters in the description | Accepted and stored as sent, useful downstream to do prompt injection/XSS type attacks |
| Principal tags | Written into every `GetScopedCredentials` event in CloudTrail |

## Recommendations

For anyone building agent harnesses on top of this:

1. Build scopes as allowlists of specific actions and resources, and never include `sts:AssumeRole`, `sts:*`, `iam:*`, `iam:PassRole`, `ssm:SendCommand` or `lambda:*` in a sub-agent's scope unless that sub-agent is meant to become someone else.
2. Leave `sts:GetScopedCredentials` out of the child's scope unless the child genuinely needs to delegate further.
3. Always set `--duration-seconds`. Without it the child lives as long as the parent.
4. Keep the CLI response out of tool stdout, transcripts and the model's context. Call the SDK and pass the credentials through the environment of the child process.
5. Use `--policy file://` (or the SDK) for the policy, and keep descriptions free of secrets.

For defenders:

1. Remove `sts:GetScopedCredentials` from roles that have no reason to delegate, or deny it with a Service Control Policy (SCP) at the AWS Organizations level. Roles with `sts:*` have it already.
2. When responding to a leaked key or session, revoke with an `aws:TokenIssueTime` deny or a deny-all on the principal. Deleting the access key alone leaves sessions and their scoped children alive.
3. Join `GetScopedCredentials` events to subsequent activity on `responseElements.credentials.accessKeyId` if you want to know which actions came from a scoped child. Alert on new `ASIA` key IDs from a session that already exists if your detections are keyed on access key ID.
4. Escape `scopedSessionDescription` before rendering it anywhere.
5. Assume principal tags are visible to everyone with trail read access, and keep sensitive data out of them.
6. The standard rules apply - any access that can lead to additional privilege must still be evaluated as is - for example, the ability to launch an EC2 instance can allow IAM privilege escalation via mounting of a privileged IAM instance role or an IAM principal having access to AssumeRole or CreateUser permissions.

## What I did not test

GuardDuty was not enabled in my test account, so I could not check whether `InstanceCredentialExfiltration` findings are caught for scoped children used off-instance, or whether GuardDuty links a scoped key back to the instance credentials it came from. The `inScopeOf` block in CloudTrail suggests the data is there. If someone runs this with GuardDuty on, do let me know and I will update the post!

I also did not test MFA-backed `GetSessionToken` sessions (whether the scoped child keeps `aws:MultiFactorAuthPresent` and can call IAM), root user sessions, IAM Identity Center sessions or SAML/web identity federated sessions.

## Final thoughts

The scope down itself held up. Through every path I tried (user sessions, IMDS role credentials, chaining, console federation) the scoped credentials could never do more than the caller could. Lifetime is capped at the parent's, the EC2 context keys survive, and session revocation based on `aws:TokenIssueTime` still covers the children.

The rough edges are in how it interacts with the rest of IAM and logging. A scope is a session policy, so anything in it that leads to another principal is a way out. Scoped children are almost invisible in CloudTrail unless you join on the access key ID, and they keep being mintable after the access key that started the chain is deleted. The description field accepts control characters and is likely to be filled from model output. All of this needs more attention once the API sits inside agent harnesses that decide on scopes at runtime.

Until next time, Happy Hacking!

## References

- AWS Security Blog, Scope down permissions of existing sessions using AWS STS GetScopedCredentials - <https://aws.amazon.com/blogs/security/scope-down-permissions-of-existing-sessions-using-aws-sts-getscopedcredentials/>
- IAM session policies - <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#policies_session>
- Revoking IAM role temporary security credentials - <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_revoke-sessions.html>
- AWS global condition context keys (`aws:TokenIssueTime`, `aws:ec2InstanceSourceVPC`, `aws:ec2InstanceSourcePrivateIPv4`) - <https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html>
- Enabling custom identity broker access to the AWS console - <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html>
- GetSessionToken API reference - <https://docs.aws.amazon.com/STS/latest/APIReference/API_GetSessionToken.html>
- curl `--aws-sigv4` - <https://curl.se/docs/manpage.html#--aws-sigv4>
- awsjail, a related post on scoping human access to `aws` - <https://ibreak.software/2026/09/awsjail-an-aws-only-login-shell-for-ssh-bastions/>
