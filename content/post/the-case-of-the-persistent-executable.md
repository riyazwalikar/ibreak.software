---
title: "The Case of the Persistent Executable"
date: 2009-07-07
categories:
- malware
- sysinternals
- windows
- case study
tags:
- process explorer

thumbnailImagePosition: left
thumbnailImage: /img/the-case-of-the-persistent-executable/7.png
---

A malware infection, sysinternal's process explorer and lots of amateurish hunting around!
<!--more-->

I woke up last Saturday around 11:00 in the morning to find my room mate sitting at our shared computer typing some document in MSWord, he then minimized the document and proceeded to open the D: drive from My Computer. My usually fast Windows responded extremely slowly to the double click. I sat bolt upright in my bed and asked him to repeat the procedure with the other drives. The same delay was noticed on the other drives too. I then asked him to right click on any drive expecting a change in the context menu due to the presence of an autorun file. The menu was intact. I then got down and sat at the chair and used the attrib command at the prompt for each drive. This is what I got.

![](/img/the-case-of-the-persistent-executable/1.jpg)

The contents of the `autorun.inf` showed me the name of the malicious executable

![](/img/the-case-of-the-persistent-executable/8.png)

I then immediately fired Process Explorer to see if the process was running. Failing to find the process or a handle to it, I then used the `attrib –s –h –r fppg1.exe` to reset attributes and proceeded to delete it using del `fppg1.exe`. I repeated the same procedure with the `autorun.inf` file. Since I have 6 partitions on my hard drive, I wrote a bat file, named it `clean.bat` and saved it in `%systemroot%` with the following contents.

```
@echo off
attrib -s -h -r fppg1.exe
del fppg1.exe
attrib -s -h -r `autorun.inf`
del `autorun.inf`
echo All done
echo.
```

I then ran `clean.bat` from the console on each partition. Happy that my system was back to normal, I restarted explorer to remove the effects of the `autorun.inf` file on the default open option on the drives. I then proceeded to open F: drive using the double click through My Computer. I was surprised to see the delay occurring again. The attrib command confirmed my doubts. The two files were back. I decided to dump the strings from the `fppg1.exe` file to see if I could find any clues. I ran the strings utility and piped the output to a text file called `fppg1.text`.

![](/img/the-case-of-the-persistent-executable/2.png)

The file contained loads of ASCII characters and just three APIs that I recognized. That didn’t help much.

![](/img/the-case-of-the-persistent-executable/3.png)

I then fired up Sysinternals' Process Monitor to see what process was writing these files to disk. I used two filters with Path contains `autorun.inf` then include and Path contains fppg1.exe then include. I was surprised to see which process was writing, setting attributes and querying information.

![](/img/the-case-of-the-persistent-executable/4.png)

I then right clicked on Explorer to view its stack when `IRP_MJ_CREATE` Operation was performed. The stack had one unfamiliar entry.

![](/img/the-case-of-the-persistent-executable/5.png)

I used the find handle or dll feature of Process Explorer to search for `amvo0.dll`. The returned results didn’t raise my spirits.

![](/img/the-case-of-the-persistent-executable/6.png)

The dll had attached itself to other processes I had opened after restarting explorer. I then opened up cmd, changed to `C:\Windows\System32\` and used the attrib command to confirm my suspicions about the attributes of `amvo0.dll`. I wasn’t disappointed.

![](/img/the-case-of-the-persistent-executable/7.png)

I suspected that there could be an associated executable also present in the same directory and hence used `attrib amv*`. With my suspicions confirmed, I used `strings.exe` to dump strings from `amvo.exe` and did a file compare with `fppg1.txt`. Bingo! They were the same files in essence. The dll `amvo0.dll` was making `explorer.exe` and the system process to recreate the files `fppg1.exe` and `autorun.inf` whenever they were not found in the root of the drives. I used attrib again to remove the system and hidden attribute from `amvo.exe` and `amvo0.dll` and deleted `amvo.exe` through the command prompt. The file `amvo0.dll` was in memory and hence could not be deleted. One shortcoming of Process Explorer, I found would have really helped me, was to unload dlls which would have allowed me to delete the file immediately. I used `autoruns.exe`, another of Sysinternals creations, and found that `amvo.exe` created a registry entry in `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` that caused it to be run at system startup. With the file gone, I restarted my system and then deleted `amvo0.dll` manually, `fppg1.exe` and the `autorun.inf` files using the bat file.

Case closed. I then went on to start my morning.

---