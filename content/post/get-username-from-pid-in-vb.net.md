---
title: "Get username from PID in VB.NET"
date: 2015-12-03
categories:
- windows api
tags:
- openprocess
- openprocesstoken
- gettokeninformation
- convertsidtostringsid

thumbnailImagePosition: left
thumbnailImage: /img/get-username-from-pid-in-vb.net/1.png
---

A reusable function that can be used to obtain the username given a Process ID on Windows. Code is in VB.NET.

<!--more-->

## Introduction

As part of a larger program that I'm writing to work with Windows processes, threads and window objects, I wrote a function that allows you to get the username of the user who owns a given process. As simple as it sounds, there is no straightforward way of obtaining the username given the process id in Windows programmatically.

## What is required?

We could use a WMI query to get the username from a PID but that is painfully slow especially if you plan on doing enumeration for all the processes running on your system. The faster way is to use the Win32 API. The primary APIs that are needed to achieve this are:

- [OpenProcess](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684320(v=vs.85).aspx)
- [OpenProcessToken](https://msdn.microsoft.com/en-us/library/windows/desktop/aa379295(v=vs.85).aspx)
- [GetTokenInformation](https://msdn.microsoft.com/en-us/library/windows/desktop/aa446671(v=vs.85).aspx)
- [ConvertSidToStringSid](https://msdn.microsoft.com/en-us/library/windows/desktop/aa376399(v=vs.85).aspx)

### What is the flow?

The flow would be as follows:

1. Using `OpenProcess` with the `PROCESS_QUERY_INFORMATION` access right, you can open a handle to a process. This will fail if you do not have the necessary access levels. For example, a limited user using querying a process owned by `NT Authority\System`
2. Once you have the handle, you can call `OpenProcessToken` requesting the `TOKEN_QUERY` access right. This function returns a pointer to a handle that identifies the newly opened access token
3. Once you have obtained the handle to the access token, calling `GetTokenInformation` with the access token handle returns the length of the buffer for the `TokenInformation` parameter when a null `TokenInformation` value is passed
4. The `GetTokenInformation` function has to be called again with the length of the buffer obtained from the previous call along with a pointer to the buffer that will be populated via the `TokenInformation` parameter. The `TokenInformation` structure can then be cast to a structure that is defined as `TOKEN_USER` that contains a structure called `SID_AND_ATTRIBUTES` that defines the SID that represents the user associated with the access token that was passed to `GetTokenInformation`
5. Once the `TokenUser` variable is populated, we can use the `ConvertSidToStringSid` function which takes in a SID structure and provides a readable SID value
6. This SID value can then be passed to `Principal.SecurityIdentifier` to be translated to a username using the `Principal.NTAccount` type


## The Code

The following code can be used as a module in your projects.

```vb
Imports System.Runtime.InteropServices
Imports System.Security
Imports System.ComponentModel

Module ProcessFunctions
 <DllImport("kernel32.dll")> _
 Public Function OpenProcess(processAccess As ProcessAccessFlags, bInheritHandle As Boolean, processId As Integer) As Integer
 End Function

 <DllImport("advapi32.dll", SetLastError:=True)> _
 Public Function OpenProcessToken(ByVal ProcessHandle As IntPtr, ByVal DesiredAccess As Integer, ByRef TokenHandle As IntPtr) As Boolean
 End Function

 <DllImport("advapi32.dll", SetLastError:=True)> _
 Public Function GetTokenInformation(ByVal TokenHandle As IntPtr, ByVal TokenInformationClass As TOKEN_INFORMATION_CLASS, _
 ByVal TokenInformation As IntPtr, ByVal TokenInformationLength As System.UInt32, ByRef ReturnLength As System.UInt32) As Boolean
 End Function

 <DllImport("advapi32.dll", SetLastError:=True)> _
 Public Function ConvertSidToStringSid(ByVal psid As IntPtr, ByRef pStringSid As String) As Boolean
 End Function

 <DllImport("kernel32.dll", SetLastError:=True)> _
 Public Function GetExitCodeProcess(ByVal hProcess As IntPtr, ByRef lpExitCode As System.UInt32) As Boolean
 End Function

 Public Structure SID_AND_ATTRIBUTES
   Public Sid As IntPtr
   Public Attributes As Int32
 End Structure

 Public Structure TOKEN_USER
   Public User As SID_AND_ATTRIBUTES
 End Structure

 Enum ProcessAccessFlags
   All = &H1F0FFF
   Terminate = &H1
   CreateThread = &H2
   VirtualMemoryOperation = &H8
   VirtualMemoryRead = &H10
   VirtualMemoryWrite = &H20
   DuplicateHandle = &H40
   CreateProcess = &H80
   SetQuota = &H100
   SetInformation = &H200
   QueryInformation = &H400
   QueryLimitedInformation = &H1000
   Synchronize = &H100000
 End Enum

 Enum TOKEN_INFORMATION_CLASS
   TokenUser = 1
   TokenGroups
   TokenPrivileges
   TokenOwner
   TokenPrimaryGroup
   TokenDefaultDacl
   TokenSource
   TokenType
   TokenImpersonationLevel
   TokenStatistics
   TokenRestrictedSids
   TokenSessionId
   TokenGroupsAndPrivileges
   TokenSessionReference
   TokenSandBoxInert
   TokenAuditPolicy
   TokenOrigin
 End Enum

Public Function GetUsernameFromProcessID(pid As Long) As String
  Dim phnd As Integer = OpenProcess(ProcessAccessFlags.QueryInformation, False, pid)
  If phnd <> 0 Then
    Dim Thnd As Integer
    OpenProcessToken(phnd, Principal.TokenAccessLevels.Query, Thnd)
    Dim TokenUser As TOKEN_USER
    Dim TokenInfLength As Integer = 0
    Dim sUserSid As String = ""
    Dim TokenInformation As IntPtr

    GetTokenInformation(Thnd, TOKEN_INFORMATION_CLASS.TokenUser, IntPtr.Zero, 0, TokenInfLength)
    TokenInformation = Marshal.AllocHGlobal(TokenInfLength)
    GetTokenInformation(Thnd, TOKEN_INFORMATION_CLASS.TokenUser, TokenInformation, TokenInfLength, TokenInfLength)

    TokenUser = DirectCast(Marshal.PtrToStructure(TokenInformation, TokenUser.GetType), TOKEN_USER)

    ConvertSidToStringSid(TokenUser.User.Sid, sUserSid)

    Dim username As String = New Principal.SecurityIdentifier(sUserSid).Translate(GetType(Principal.NTAccount)).ToString()
    Return username
 Else : Return "<access denied>"
 End If
End Function
End Module
```

Add this code as part of your project in a module and access the function using `GetUsernameFromProcessID(pid)` which will return the username if you have access to the process and `<access denied>` if you don't.

Happy Hacking!

---