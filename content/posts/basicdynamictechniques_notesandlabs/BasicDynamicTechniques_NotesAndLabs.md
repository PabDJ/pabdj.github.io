---
title: "Basic Dynamic Techniques - Notes & Labs"
date: 2022-12-17T12:50:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, basic, dynamic, analysis, registers]
---

## Notes
### Running malware
- How to launch DLLs successfully in dynamic analysis: take a look at the Exports with CFF Explorer and afterwards run the following command `C:\>rundll32.exe DLLname, Export arguments` with one of the Export functions that have appeared in the static analysis.
- To convert the DLL file into an executable, modify the PE header wiping the `IMAGE_FILE_DLL (0x2000)` flag from the Characteristics field in the `IMAGE_FILE_HEADER`.
- DLL malware may also need to be installed as a service, sometimes with a convenient export such as *InstallService*, as listed in *ipr32x.dll*:
`C:\>rundll32.exe ipr32x.dll, InstallService`
`C:\>net start ServiceName`

### Tools
#### Procmon
- To stop *procmon* from capturing events, choose `File ▶ Capture Events`.
- Before using *procmon* for analysis, first clear all currently captured events to remove irrelevant data by choosing `Edit ▶ Clear Display`.
- Regarding filters, the most important ones for malware analysis are *Process Name*, *Operation*,
and *Detail*.

#### Process Explorer
- When you double-click a process name, a window is displayed. Inside the *Image* tab, there is an interesting option which is *Verify*. It compares the image of the process that is loaded in memory with the one that is stored on the disk, so we can be sure if it has been corrupted.
- Another way of comparing processes is through *Strings* tab.
- To check if a process uses a malicious DLL we can use the tool `Find ▶ Find Handle or DLL`.

#### Regshot
- Click *1st Shot*, detonate the malware sample and after waiting some time click *2nd Shot* and *Compare*.
- Be aware of some noise because the random-number generator seed is constantly updated in the registry.

#### ApateDNS
- If your app is not working properly, check [this video](https://www.youtube.com/watch?v=mqXayCkFaKY).
- Gateway will be *localhost* or *REMnux machine*, depending on if we need to keep running a fake web server.

#### INetSim
- Keep it running on the background and check the logs after the malware has been executed.


## Labs

### Lab 1

**1. What are this malware’s imports and strings?**

The only import that can be observed is ExitProcess from kernel32.dll. This fact already shows suspicious activity but with PEiD we can confirm that the malware sample is packed (PEncrypt 3.1)

Regarding the strings, they are the following:
{{< figure src="../3-1.PNG" >}}

**2. What are the malware’s host-based indicators?**

In order to check the manipulations that the malware performs in the victim machine, we run ApateDNS, Procmon, Process Explorer, RegShot and Wireshark . Once all the tools are running, we execute the malware sample with administrator account.
After several tries, it seems that this lab does not work on Windows 7, so we will have to try on Windows XP.

Results obtained with Regshot:
{{< figure src="../3-1-2.PNG" >}}

Wireshark: the data that is sent is not legible and is different between streams.
{{< figure src="../3-1-3.PNG" >}}

ApateDNS: the malware tries to connect to the domain www[.]practicalmalwareanalysis[.]com
{{< figure src="../3-1-4.PNG" >}}

Procmon: it creates the file *vmx32to64.exe* and the registry entry "*HKLM\Software\Microsoft\Windows\CurrentVersion\Run\VideoDriver*" apart from several DLL files.
{{< figure src="../3-1-5.PNG" >}}

**3. Are there any useful network-based signatures for this malware? If so, what are they?**
We observed in the strings the presence of the URL *www[.]practicalmalwareanalysis[.]com* and in fact the malware tried to connect to that URL.


### Lab 2

**1. How can you get this malware to install itself?**

After checking the exports from the .dll file we know that it has to be installed as a Service.
{{< figure src="../3-2.PNG" >}}

We have to run in the console:
`C:\>rundll32.exe Lab03-02.dll, installA`

**2. How would you get this malware to run after installation?**

I tried to run the following command `C:\>net start ServiceMain` but nothing happened. I had to check the solution because I didn't know which was the name of the service. It is true that one of the strings was *IPRIP*, but I would have never thought that was the correct name of the service. In fact, I thought it was junk strings.

Never mind, with `C:\>net start IPRIP` we can see how the magic goes.

**3. How can you find the process under which this malware is running?**

With the strings examination we observed several references to "*svchost.exe*" so we can agree that we should monitor this process.

After running the sample together with our tools we can confirm that "*svchost.exe*" is linked with the malicious DLL. If we use `Find ▶ Find Handle or DLL` from Procmon we can observe the link.
{{< figure src="../3-2-1.PNG" >}}

**4. Which filters could you set in order to use procmon to glean information?**

The key filter is the one that allow us to select the events by PID. In our case we retrieved all the information from the process 1084 which is the one linked to "*svchost.exe*".

**5. What are the malware’s host-based indicators?**

The malware created the file "*net1.exe*" at "*C:\Windows\system32*" and it is playing around with registry entries related to Internet as it can be observed in the following image. The same information was also collected with RegShot.
{{< figure src="../3-2-3.PNG" >}}

**6. Are there any useful network-based signatures for this malware?**
As we can observe with ApateDNS, the malware tries to connect to "www[.]practicalmalwareanalysis[.]com"
{{< figure src="../3-2-2.PNG" >}}


### Lab 3

**1. What do you notice when monitoring this malware with Process Explorer?**

The process ends its execution after a few seconds. Before that, it creates another process "*svchost.exe*".

**2. Can you identify any live memory modifications?**

With Process Explorer we can compare the strings that are loaded in memory and in disk, and we can observe that they are different.

|     svchost.exe Image Strings     |    svchost.exe Memory Strings     |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../3-4-1.PNG" >}} | {{< figure src="../3-4-2.PNG" >}} |

**3. What are the malware’s host-based indicators?**

The strings shown by "*svchost.exe*" in memory mentioned a file called "*practicalmalwareanalysis.log*".

**4. What is the purpose of this program?**

The strings showed some references to keystrokes such as *ENTER*, *BACKSPACE*, etc. After collecting events with Procmon, I restarted the capture, and it was visible that the file take everything that was typed in the notepad to that file.
{{< figure src="../3-4-3.PNG" >}}

{{< figure src="../3-4-5.PNG" >}}


### Lab 4

**1. What happens when you run this file?**

It starts a command line process ("*cmd.exe*") and a few seconds later it ends. Apart from that, when you try to run it again, it deletes itself.

**2. What is causing the roadblock in dynamic analysis?**

After taking a look at the strings, there may be an error during the execution like missing arguments. There are some prints from the code that show that we are covering an unexpected path.
{{< figure src="../3-3.PNG" >}}

**3. Are there other ways to run this program?**

According to the solution, this question can be answered in future chapters.