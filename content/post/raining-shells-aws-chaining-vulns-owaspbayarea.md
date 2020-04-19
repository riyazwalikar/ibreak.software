---
title: "Raining shells in AWS by chaining vulnerabilities - OWASP Bay Area Meetup"
date: 2019-08-12
categories:
- cloud
- aws
- penetration testing
- offsec
tags:
- shells
- exploitation
- iam
- aws

thumbnailImagePosition: left
thumbnailImage: /img/raining-shells-aws-chaining-vulns-owaspbayarea/1.png
---

Slides of my talk on using mis-configurations, overtly permissive IAM policies and application security vulnerabilities to get shells in AWS EC2 instances and go beyond the plane of attack. Presented at OWASP Bay Area August 2019 meetup.

<!--more-->

The talk covers 3 scenarios that were built using real world cases of penetration testing exercises that led to shell access and access to data beyond the EC2 instances that were compromised.

The presentation contains commands and example output for all the scenarios covered.

3 Scenarios covered are

- Case 1: Misconfigured bucket to system shells
- Case 2: SSRF to Shell via IAM Policies
- Case 3: Client-Side Keys, IAM Policies and a Vulnerable Lambda

<script async class="speakerdeck-embed" data-id="495e050b1ded49daa0cc33e413fb81ec" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

---