---
title: "Basic Static Techniques - Notes & Labs"
date: 2022-12-08T12:50:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, basic, static, analysis, unpacking]
---

## Notes
#### Hashing: A Fingerprint for Malware
We will use **HashCalc** tool.

#### Finding Strings
For this task we will use FLOSS (FLARE Obfuscated String Solver) tool 

#### Packed and Obfuscated Malware
- Packed and obfuscated code will often include at least the functions **_LoadLibrary_** and **_GetProcAddress_**, which are used to load and gain access to additional functions.
- [UPX: the Ultimate Packer for eXecutables](https://upx.github.io/)
- Several Microsoft Windows functions allow programmers to import linked functions not listed in a program's file header. Of these, the two most commonly used are **_LoadLibrary_** and **_GetProcAddress_**. **_LdrGetProcAddress_** and **_LdrLoadDll_** are also used.

{{< figure src="../CommonDLLs.png" title="Common DLLs" >}}

- When Microsoft updates a function and the new function is incompatible with the old one, Microsoft continues to support the old function. The new function is given the same name as the old function, with an added Ex suffix.
- Many functions that take strings as parameters include an A or a W at the end of their names. This letter does not appear in the documentation for the function; it simply indicates that the function accepts a string parameter and that there are two different versions of the function: one for ASCII strings and one for wide character strings.
- **_SetWindowsHookEx_** is commonly used in spyware and is the most popular way that keyloggers receive keyboard inputs

## Labs

### Lab 1

**2. When were these files compiled?**
We can observe the exact date with CFF Explorer tool. Lab01-01.exe and Lab01-01.dll were compiled on 2010/12/19 Sunday 16:16 UTC

**3. Are there any indications that either of these files is packed or obfuscated? If so, what are these indicators?**
One way to detect packed files is with the PEiD program. In this case, both files do not seem to be packed.

**4. Do any imports hint at what this malware does? If so, which imports are they?**
With CFF Explorer we can look at Lab01-01.exe imports. In the following picture we can observe them:
{{< figure src="../1-4.png" >}}

As we are dealing with a DLL file, the most common thing would be that the malware is a rootkit. In this case, the .exe file tries to substitute the authentic DLL with a malicious one. For this task, the .exe imports some functions such as FindFirstFile, FindNextFile, CreateFile, CreateFileMapping, MapViewOfFile and UnmapViewOfFile.

**5. Are there any other files or host-based indicators that you could look for on infected systems?**
FLOSS tool provides the strings linked to the .exe file. Thanks to that we can observe that the malware tries to hide the malicious DLL playing with the similarity between letter “l” and number “1”. Its name is kerne132.dll instead of kernel32.dll.

**6. What network-based indicators could be used to find this malware on infected machines?**
The .dll contains an IP address (127.26.152[.]13) which could be useful for detecting if other machines make connections to that IP. In this case it is a localhost IP.

**7. What would you guess is the purpose of these files?**
Rootkit.


### Lab 2

**2. Are there any indications that this file is packed or obfuscated? If so, what are these indicators? If the file is packed, unpack it if possible.**
When we try to load the file into CFF Explorer, the file info shows UPX v3.0 which is a famous packer. In fact, this packer was mentioned in the chapter.

For unpacking it we just have to run the following command: `upx -d Lab01-02.exe`

**3. Do any imports hint at this program’s functionality? If so, which imports are they and what do they tell you?**
{{< figure src="../2-3.png" >}}

The sample imports WININET library which is used for implementing higher-networking protocols such as HTTP. We have to take into account that the sample is packed so guessing program’s functionality is not straightforward.

**4. What host- or network-based indicators could be used to identify this malware on infected machines?**
After unpacking we can observe that the malware sample connects to the following URL: hxxp://malwareanalysisbook[.]com and it creates a process called MalService.


### Lab 3

**2. Are there any indications that this file is packed or obfuscated? If so, what are these indicators? If the file is packed, unpack it if possible.**
We can observe that the file is packed with FSG 1.0 dulek/xt.

As we have not covered yet the process of unpacking malware samples that have not been packed with UPX we are not supposed to know this.

**3. Do any imports hint at this program’s functionality? If so, which imports are they and what do they tell you?**
As we have not unpacked the sample, we cannot make further analysis.

**4. What host- or network-based indicators could be used to identify this malware on infected machines?**
As we have not unpacked the sample, we cannot make further analysis.


### Lab 4

**2. Are there any indications that this file is packed or obfuscated? If so, what are these indicators? If the file is packed, unpack it if possible.**
The file does not seem to be packed.

**3. When was this program compiled?**
The program was compiled on Friday 30th August 2019 at 22:26

Compilation times can be faked, and I think that this is the case as _Practical Malware Analysis_ book was written in 2011. Malware sample from the future? 😉

**4. Do any imports hint at this program’s functionality? If so, which imports are they and what do they tell you?**
After observing some imports we can think that we are dealing with a sample that performs process injection. This hypothesis is supported by the different imports from the program such as _GetCurrentProcess_, _GetProcessAddress_, _OpenProcess_ and _CreateRemoteThread._

**5. What host- or network-based indicators could be used to identify this malware on infected machines?**
If we take a look at the strings, we can see some malicious programs such as _winup.exe_ and _wupdmgr.exe_ and malicious URLs such as hxxp://practicalmalwareanalysis[.]com/updater.exe

**6. This file has one resource in the resource section. Use Resource Hacker to examine that resource, and then use it to extract the resource. What can you learn from the resource?**
After taking a look again at the imports we can see that there are some suspicious calls such as *FindResourceA*, *LoadResource* and *SizeofResource*.

I have learnt that with ResourceHacker we can convert the binary data to another .exe file and then perform another analysis to that sample. As it is stated in the solutions, the original file brings another sample attached, which is the one that downloads more malware from the URLs that we found before (it contains *suspicious* imports like *URLDownloadToFile*.