---
title: "XSPA / SSRF bug with Facebook’s Developer Web Application"
date: 2013-05-10
categories:
- bugbounty
- research
- appsec
- cross site port attacks
tags:
- xspa
- ssrf
- facebook

thumbnailImagePosition: left
thumbnailImage: /img/xspa---ssrf-bug-with-facebook-s-developer-web-application/1.png
---

The first XSPA/SSRF bug that led to the discovery of this issue in other applications and eventually a paper that was presented at multiple conferences.

<!--more-->

## Background

This post is about a responsible disclosure I made to Facebook recently about a vulnerability with their `developer.facebook.com` web application that allowed an attacker to perform port scans on remote machines on the Internet. An attacker could scan Internet facing machines for open ports and proxy his scans through Facebook’s IP addresses using this vulnerability.

## The flaw

The URL at `http://developers.facebook.com/tools/debug/og/object` is vulnerable to a SSRF / XSPA vulnerability (CWE-918), via the 'q' parameter, allowing an attacker to port scan external internet facing systems and identify IP addresses on the Internal network as well based on error messages and response data.

The following steps can be used to reproduce the issue. All scans have been verified against `scanme.nmap.org` that is known to have ports 22, 80 and 9929 open.

### Step 1

- Navigate to `http://developers.facebook.com/tools/debug` and enter the following URL in the 'q' text box (open port test). 
  - `http://scanme.nmap.org:22/index.html`

A GET request is sent to `http://developers.facebook.com/tools/debug/og/object` with the 'q' parameter being passed via the URL.

### Step 2

- Notice the "Response Code" received by Facebook from the remote server (502).

![](/img/xspa---ssrf-bug-with-facebook-s-developer-web-application/1.png)

### Step 3

- Repeat the request for `http://scanme.nmap.org:25/index.html` (closed port test)

### Step 4

- Notice the Response code received by facebook from the remote server (503).

![](/img/xspa---ssrf-bug-with-facebook-s-developer-web-application/2.png)


Other responses that have been noticed for open ports include 200, 404 and 206. An error response "Error parsing input URL, no data was scraped." is also seen if a request to a non ASCII port is sent (3389 etc).


![](/img/xspa---ssrf-bug-with-facebook-s-developer-web-application/3.png)

I wrote a simple port scanner in python that utilises this bug to make connections to remote systems which you can [download from Github](https://github.com/riyazwalikar/xspafbportscanner).

![](/img/xspa---ssrf-bug-with-facebook-s-developer-web-application/4.png)


Facebook paid out a bounty for this bug although they did not fix the issue completely. This is what they had to say: 

> "There are quite a few ways that our service can be made to issue requests to third-parties, and it's unfortunately not feasible to block non-standard ports on all of them. This debugging tool is one of those endpoints where it's incredibly helpful to allow requests to non-standard ports. We monitor and rate-limit the usage of this endpoint but the implications here are low-risk enough that we've decided not to eliminate this helpful functionality entirely."

And I agree, rate-limiting the number of requests received from a single IP/subnet/network that look suspicious is one way this could be kept under check, however this bug will remain a good example of a functionality that can be heavily abused. 

Happy hunting!! 

---