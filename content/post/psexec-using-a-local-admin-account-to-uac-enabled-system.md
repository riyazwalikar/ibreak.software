---
title: "psexec using a local admin account to a UAC enabled system"
date: 2016-02-20
categories:
- psexec
tags:
- windows registry
- windows
- uac
- metasploit

thumbnailImagePosition: left
thumbnailImage: /img/psexec-using-a-local-admin-account-to-uac-enabled-system/1.png
---

Enabling the abililty to use psexec over the network when credentials are available by toggling a value in the Windows registry.

<!--more-->

## Introduction

To protect users across the network, [Windows UAC](http://windows.microsoft.com/en-in/windows/what-is-user-account-control) imposes token restrictions on local administrators logging in via the network (using the `net use \\computer\\c$` share for example). This means that a local administrator will not be able to perform administrative tasks and will not have the ability to elevate to full admin rights over the network, even when credentials are available.

This works well if you are securing systems. However, during a pentest, hash/password reuse via psexec for example, will fail. Simply because connecting to the `C$ admin` share to run the psexec service will fail. My friend and systems hacker [Anant Shrivastava](https://twitter.com/anantshri) pointed this out during some testing that he was doing, prompting me to blog about this.

## Setup and testing this

I setup a Windows 7 machine with UAC enabled, an administrative account called `testadmin` with password `testadmin` and used the `exploit/windows/smb/psexec` exploit module from metasploit to test this in a lab environment and saw the following error:

![](/img/psexec-using-a-local-admin-account-to-uac-enabled-system/1.png)

[Microsoft recommends a registry edit to disable UAC remote restrictions](https://support.microsoft.com/en-us/kb/951016). To make this change, follow these steps:

1. Open the registry editor using the `regedit` command via Start > Run
2. Navigate to `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
3. In the right pane, if the `LocalAccountTokenFilterPolicy` DWORD value doesn't exist, create it.
4. Set its value to 

![](/img/psexec-using-a-local-admin-account-to-uac-enabled-system/2.png)

The changes take effect immediately. I tried the Metasploit exploit again and voila it worked this time:

![](/img/psexec-using-a-local-admin-account-to-uac-enabled-system/3.png)

This registry change allows Sysinternals Psexec utility to function as well apart from other utilities that require a privileged token on the C$ share (or any other admin share).

Happy Hacking!

---