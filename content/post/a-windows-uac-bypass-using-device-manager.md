---
title: "A Windows UAC Bypass using Device Manager"
date: 2017-05-18
categories:
- windows
- post exploitation
- uac
tags:
- poc

thumbnailImagePosition: left
thumbnailImage: /img/a-windows-uac-bypass-using-device-manager/1.png
---

A simple UAC bypass to launch programs using the Device Manager on Windows 10. Requires access to GUI. Limited usage but fun conceptually.

<!--more-->

## Introduction

Today while working on a Windows 10 machine, I had the need to open the Device Manager for some hardware maintenance. While opening the Windows Device manager I noticed that there was no UAC prompt when I started it. This was a little strange because the Device Manager exists as a Management Console snap-in in `%systemroot%\System32\devmgmt.msc` and is launched by `mmc.exe`.


## What's happening under the hood

When you start the Device Manager, `mmc.exe` is launched with `%systemroot%\System32\devmgmt.msc` as an argument.

![](/img/a-windows-uac-bypass-using-device-manager/1.png)

Independently, the Microsoft Management Console requires elevation to run. When you go to run and launch mmc.exe, Windows will ask you to allow elevation using the UAC prompt.

![](/img/a-windows-uac-bypass-using-device-manager/2.png)

If the Device Manager can be opened without a UAC prompt it means that it was launched with [High Integrity level](https://msdn.microsoft.com/en-us/library/bb625963.aspx). If this process can be used to spawn other processes, they will be launched with High integrity level too.

![](/img/a-windows-uac-bypass-using-device-manager/3.png)

It was trivial from here to find a way to open an elevated Command Prompt without being prompted by UAC.
User Access Control on my machine was set to the default value of "Notify me only when apps try to make changes to my computer (default)"

![](/img/a-windows-uac-bypass-using-device-manager/4.png)

To reproduce this:

- Go to Start run and type devmgmt.msc. Notice that there is no UAC prompt.
- Once the Device Manager opens, goto Help > Help Topics

![](/img/a-windows-uac-bypass-using-device-manager/5.png)

- This will open up the Microsoft Management Console Help window. Right click any where in the right pane and select View Source

![](/img/a-windows-uac-bypass-using-device-manager/6.png)

- This will open notepad (the editor may vary if you have set IE's source viewer to a different text editor).
- Using notepad's File > Open menu, navigate to the System32 directory.
- Set the File type to "All files (*.*)" and right click select "Run as Administrator" on `cmd.exe`

![](/img/a-windows-uac-bypass-using-device-manager/7.png)

- This should launch a elevated Command Prompt without any UAC prompts. You can run `whoami /priv` to check that you have a lot of privileges available now (disabled but available).
- The process tree also shows that the `cmd.exe` that was spawned was started with High Integrity level.

![](/img/a-windows-uac-bypass-using-device-manager/8.png)

There are several other techniques available on Windows that allow you to bypass UAC and most of them are well documented (see references below).
Please note, according to Microsoft, [UAC bypasses are not a security problem as UAC is a convenience feature (more references in that page)](https://en.wikipedia.org/wiki/User_Account_Control#Security).

### Other UAC Bypass References

1. https://enigma0x3.net/2016/08/15/fileless-uac-bypass-using-eventvwr-exe-and-registry-hijacking/
2. https://enigma0x3.net/2016/07/22/bypassing-uac-on-windows-10-using-disk-cleanup/
3. https://github.com/hfiref0x/UACME
4. https://habrahabr.ru/company/pm/blog/328008/ [Use Google translate, worth reading]

Happy Hacking!

---