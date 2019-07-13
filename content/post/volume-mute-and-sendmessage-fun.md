---
title: "Volume Mute and SendMessage() Fun"
date: 2015-11-09
categories:
- windows api
tags:
- sendmessage
- mute

thumbnailImagePosition: left
thumbnailImage: /img/volume-mute-and-sendmessage-fun/1.png
---

Small piece of code written in .NET to create a binary that when run will mute the speaker. Uses Windows API (SendMessage).

<!--more-->

As part of managing the [@wincmdfu](https://twitter.com/wincmdfu) twitter account, I try and find alternate ways of doing stuff in Windows using the commandline. I normally have a bunch of batch files, VB script files and powershell cmdlets in a custom folder in `%PATH%` that I call when required. 

While listening to some music the other day, I realized there is no shortcut that I know (or have discovered yet) that allows me to mute the audio on my system. I have a hardware key on the keyboard that mutes the volume, but no command to do this (remotely for example - which is cooler).

Enter the [Windows SendMessage](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644950(v=vs.85).aspx) API. This is a fantastic API that Windows uses to send messages to window objects given the handle of the target window (amongst several other things). This window could be in a different process as well. The SendMessage function calls the window procedure for the specified window and does not return until the window procedure has processed the message.

`SendMessage` can be used to send the `WM_APPCOMMAND` with the `APPCOMMAND_VOLUME_MUTE` parameter to the current window handle to toggle mute. This can programmatically be put into a Form/console application and run from command line. Here's my VB.NET code:


```
Public Class Form1
    <Runtime.InteropServices.DllImport("user32.dll")> _
    Public Shared Function SendMessageW(ByVal hWnd As IntPtr, ByVal Msg As Integer, ByVal wParam As IntPtr, ByVal lParam As IntPtr) As IntPtr
    End Function

    Private Const APPCOMMAND_VOLUME_MUTE As Integer = &H80000
    Private Const WM_APPCOMMAND As Integer = &H319

    Private Sub Main_Load(sender As Object, e As EventArgs) Handles MyBase.Load
        SendMessageW(Me.Handle, WM_APPCOMMAND, Me.Handle, APPCOMMAND_VOLUME_MUTE)
        End
    End Sub
End Class
```

Compile this into an executable called `mute.exe`, store it in `%PATH%` and you are done!

Happy Hacking!

---