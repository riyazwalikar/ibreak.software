---
title: "The Case of the Intelligent Spambot"
date: 2009-07-22
categories:
- malware
- sysinternals
- windows
- case study
tags:
- process explorer
- process monitor

thumbnailImagePosition: left
thumbnailImage: /img/the-case-of-the-intelligent-spambot/3.png
---

A malware that used NTFS Alternate Data Streams and Windows services to send spam on the Internet.
<!--more-->

I woke up last Sunday to my friend’s unforgivable rumblings about the Internet Speed sitting at the lone desktop in our room. I quickly ducked my head under the covers, what would I expect from a 128 Kb line, 13 or maybe 14 Kbps transfer speed? I tried to convince him half sleepily that it was natural, but when he said he had been downloading a 642 KB Word Document for the past 15 minutes, I had to sit up in bed. We never had a speed issue with the ISP, was hoping this was a first.

I quickly opened the Windows Task Manager, expecting to see any unheard process trying to steal my bandwidth. The processes didn’t look funny, at least not all of them, because I hardly have 15 different processes running on my system. Remember, this was Windows XP and has such did not have a lot going on in terms of processes. But two things stood out. There were an unexpected number of `svchost.exe` running and a process called `rs32net.exe`. I quickly fired up Process Explorer for a more detailed look and to see what new services were being run. The Process Monitor’s Image tab of the Properties for `rs32net.exe` quickly confirmed my suspicions about its intentions.

![](/img/the-case-of-the-intelligent-spambot/1.png)

I had seen files like these before. Although I wasn’t sure what was eating my bandwidth yet. I fired up the prompt and type `netstat -a`. I almost fainted at the speed at which the screen went by. My computer was trying to connect to the SMTP ports of several systems online! Here’s a snipped output from netstat.

![](/img/the-case-of-the-intelligent-spambot/2.png)

I needed professional help. I quickly fired up `TCPView` from Sysinternals. Looking at the hundreds of connections being requested did not hurt as much as knowing which processes were responsible.

![](/img/the-case-of-the-intelligent-spambot/3.png)

I had to finally agree that my system was infected by a spambot. I still couldn’t figure out how `rs32net.exe` fit into the picture. So I simply killed it to take a look at it later. The spambot was connecting first to multiple HTTP servers, presumably to download a list of addresses to which it had to attempt connection. It then proceeded to connect to the several computers that were visible in the list in `TCPView`.

I first checked the strings in memory for one of the svchost processes and saved them for later analysis. The strings that I saw were definitely not part of svchost.

![](/img/the-case-of-the-intelligent-spambot/4.png)

Since svchost was involved, I opened up the Windows Services Management Console and checked what services where running on my system. Two services looked out of place, without any descriptions, and set to automatic yet not started.

![](/img/the-case-of-the-intelligent-spambot/5.png)

I had never heard of them before, decided to take a look at their properties to determine what executable it was. Was surprised to see a colon in the filename of the executable. The `ext.exe` that comes after the svchost is an NTFS ADS or Alternate Data Stream. To access an ADS you have to specify the "host" file followed by a colon and the filename of the ADS. By default all files on an NTFS partition have an ADS, the data in the file itself is stored as an ADS without any name as `filename::$DATA`

So I now knew the exact file that was causing the issue. I used `Streams` from Sysinternals to view the ADS size. `Streams` provides you an option to delete the ADS but I wanted to extract it.

![](/img/the-case-of-the-intelligent-spambot/6.png)

To extract the ADS, I used a tool that I had written in college for a software competetion, called `NTStream`. I ran the tool on the whole Windows directory. `NTStream` scanned 2318 directories and 26802 files in under a minute and displayed the search result in its ListView Interface.

![](/img/the-case-of-the-intelligent-spambot/7.png)

I extracted the stream using the tool and saved it for analysis.

I then proceeded to delete the `FCI` and the `ICF` services entry from the Windows Registry under the `HKLM\System\CurrentControlSet\Services\FCI` and `HKLM\System\CurrentControlSet\Services\ICF` respectively. I then deleted `rs32net` from the `System32` directory, just in case, and then proceeded to enable my Network Connection that I had disabled to prevent the ISP calling up or disconnecting me forever from the online world for using my system to spam.

As soon as I enabled the Internet Connection, the flurry of Network Activity was reported again by `TCPView`. Damn! This was definitely the sign of an executable in memory or some driver or dll that was still doing what it was supposed to do, spam.

I fired up `Procmon` to view disk activity just in case to see if svchost was acting funny elsewhere on the hard drive. Never before in my life was I greeted with this by `Procmon`.

![](/img/the-case-of-the-intelligent-spambot/8.png)

I was sure I was an Administrator on my system. Very sure about it. I checked just in case wondering whether I was downgraded to a limited user by the infection. The command `net user %username%` at a command prompt told me I was very well an Administrator and a member of the Debuggers Group. I ran `Procmon` again and viewed its properties in Process Explorer to make sure I had the `SeLoadDriverPrivelege` enabled. I very well had.

![](/img/the-case-of-the-intelligent-spambot/9.png)

Assuming a rootkit infection, I quickly pulled Sysinternals' `Rootkit Revealer` out of my toolkit and ran it, scanning in default mode. I was hoping it would return something more concrete.

The scan revealed several discrepancies, but what caught my attention was the lone result from the file system right at the bottom of the 3576 entries.

Now the only way to access the file, that I thought of was to boot from an alternate medium and check. But luckily I had installed a copy of the Recovery Console on my system. I restarted the computer in Recovery Console, navigated to the `C:\Windows\System32\Drivers` directory and issued a `dir tj*` command. I was delighted to see the file there. Although I had not exactly figured out the role of the `TJTXSGTX.SYS` driver in this whole case, I was nevertheless delighted to see that the file was at least visible under the recovery console. I promptly deleted the file and restarted the computer normally. I then enabled my Internet Connection and waited with bated breath for the sequence to start again. After 5 minutes of suspense, when nothing happened, I decided to close the case after running an `sfc /scannow`.

PS: By the way, my friend installed a popular antivirus and managed to clean other infections later in the evening. He claimed it took him little over 30 minutes, unlike me. I appreciate his humor.

---