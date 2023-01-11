---
title: "Advanced Dynamic Techniques - OllyDbg"
date: 2023-01-11T15:20:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, advanced, dynamic, analysis, x86, x64, assembly, registers, ollydbg, ghidra]
---
## Table of Contents
1. [Notes](#notes)
   1. [Memory map](#memory-map)
   2. [Breakpoints](#breakpoints)
   3. [Loading DLLs](#loading-dlls)
2. [Labs](#labs)
   1. [Lab 1](#lab-1)
   2. [Lab 2](#lab-2)
   3. [Lab 3](#lab-3)


## Notes
### Memory map
All PE files in Windows have a preferred base address, known as the image base defined in the PE header.
The image base isn’t necessarily the address where the malware will be loaded, although it usually is. Most executables are designed to be loaded at 0x00400000, which is just the default address used by many compilers for the Windows platform. Developers can choose to base executables at different addresses. Executables that support address space layout randomization (ASLR) security enhancement will often be relocated. That said, relocation of DLLs is much more common.

### Breakpoints
Hardware breakpoints are useful to protect against certain anti-debugging techniques, such as
software breakpoint scanning.

Memory breakpoints are particularly useful during malware analysis when you want to find out when a loaded DLL is used: you can use a memory breakpoint to pause execution as soon as code in the DLL is executed.

### Loading DLLs
By default, OllyDbg breaks at the DLL entry point (DllMain) once the DLL is loaded. Next, OllyDbg will pause, and you can call specific exports with arguments and debug them by selecting `Debug➡️Call DLL Export` from the main menu. In order to call exported functions with arguments inside the debugged DLL, you first need to load the DLL with OllyDbg. Then, once it pauses at the
DLL entry point, click the play button to run DllMain and any other initialization the DLL requires.

## Labs

### Lab 1

**1. How can you get this malware to install itself?**

To begin the analysis we check the imports of the file. There are some of them that stand out like *CreateFile*, *ReadFile*, *WriteFile*, *DeleteFile*, *RegSetValue*, *RegCreateKey*, *CreateService*, *ShellExecute* or *CreateProcess* among others.
{{< figure src="../9-1.PNG" >}}

Main is located at address 0x402AF0 and contains several commands such as *-in*, *-re*, *-c* and *-cc*.

|            Entrypoint             |               Main                |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../9-1-1.PNG" >}} | {{< figure src="../9-1-2.PNG" >}} |

Anyway, if we do not provide any, the malware calls a function located at 0x402B2E that checks some characters from argv buffer. It might be a password. After that, it tries to delete itself. 
{{< figure src="../9-1-3.PNG" >}}
{{< figure src="../9-1-6.PNG" >}}
{{< figure src="../9-1-4.PNG" >}}

*-in* command seems to work as an installer but if we provide it as a unique argument at OllyDbg it still does not work. In fact, it checks the existence of the registry key "*SOFTWARE/Microsoft /XPS*" and if the call is not successful it tries to delete itself again.

The beginning of the program can be summarized with the following screenshot:
{{< figure src="../9-1-5.PNG" >}}

We have different ways in order to run the malware. The first approach could be to patch the result of the call that checks the registry key but this could lead to more problems in the future, so it would be better to patch the *CheckPassword* function or even provide the password as we already know from the screenshots above that it is "*abcd*".

#### Patching *CheckPassword* function
The password is located at 0x402B2E. If the password is correct, the function return 1 at EAX, otherwise EAX has value 0. In my opinion, this is the most straight forward way to patch the binary.
{{< figure src="../9-1-7.png" >}}

#### Providing arguments to the binary
We have to click Debug➡️Arguments and set: "-in abcd". This will allow us to run the binary without any problem.

**2. What are the command-line options for this program? What is the password requirement?**

There are several commands such as *-in*, *-re*, *-c* and *-cc*.

- ***-in* command**: creates a service and calls the function that achieves persistence by creating registry key.
- **-*re* command**: closes the service and removes the malware.
- **-*c* command**: updates its configuration.
- **-*cc* command**: prints the current configuration.

The password requirement can be bypassed, although it can be easily guessed as it looks for the string "*abcd*".

**3. How can you use OllyDbg to permanently patch this malware, so that it doesn’t require the special command-line password?**

Instead of modifying the EAX registry each time we reach the *CheckPassword* function, we can do the following:

`We assemble these instructions using the Assemble option in OllyDbg and get the 6-byte sequence: B8 01 00 00 00 C3. Because the CALL instruction prepares the stack, and the RET instruction cleans it up, we can overwrite the instructions at the very beginning of the password check function, at address 0x402510. Edit the instructions by right-clicking the start address you wish to edit and selecting Binary➡️Edit. We uncheck the box labeled Keep size. We then enter the assembled hex values in the HEX+06 field and click OK. Select Copy to executable➡️All modifications. Accept all dialogs, and save the new version as Lab09-01-patched.exe.`

**4. What are the host-based indicators of this malware?**

The registry key "*SOFTWARE/Microsoft /XPS*".

**5. What are the different actions this malware can be instructed to take via the network?**

The function located at 0x402020, renamed as *getcommands*, implements functionality under several commands such as *SLEEP*, *UPLOAD*, *DOWNLOAD*, *CMD* and *NOTHING*.

Sleep command as it may be obvious, sleeps for a number of seconds.
Upload (0x4019E0) connects to a socket and writes the content that has been read into a file.
{{< figure src="../9-1-8.PNG" >}}

Download (0x401870) sends the content of the file over a socket.
{{< figure src="../9-1-9.PNG" >}}

CMD that executes the shell command with cmd.exe and sends the output to the remote host.

Nothing which does nothing.

**6. Are there any useful network-based signatures for this malware?**

This question was particularly hard because although it was easy to find network-related strings such as the ones in the screenshot below, I was not able to retrieve the behaviour under it.
It is obvious that a GET request is sent under HTTP 1.0 but the messages that played with apostrophes and backticks were totally random for me. They are in fact precise messages under the C2 protocol of the malware.
{{< figure src="../9-1-10.PNG" >}}


### Lab 2

**1. What strings do you see statically in the binary?**

We can observe the imports and some interesting strings like "*www[.]practicalmalwareanalysis[.]com*" and "*ocl.exe*".
{{< figure src="../9-2.PNG" >}}

**2. What happens when you run this binary?**

It does not show much activity.

**3. How can you get this sample to run its malicious payload?**

It checks if the sample is named as "*ocl.exe*". We can rename the file or patch the value at EAX after calling strcmp.

|              Ghidra               |              OllyDbg              |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../9-2-3.PNG" >}} | {{< figure src="../9-2-4.PNG" >}} |

**4. What is happening at 0x00401133?**

The string is divided character by character into memory in order to obfuscate it.

**5. What arguments are being passed to subroutine 0x00401089?**

The string *1qaz2wsx3edc* and a pointer to an array.
{{< figure src="../9-2-5.PNG" >}}

**6. What domain name does this malware use?**

The domain is  "*www[.]practicalmalwareanalysis[.]com*". Fortunately [FLOSS](https://github.com/mandiant/flare-floss) was able to decode it without any effort.
{{< figure src="../9-2-6.PNG" >}}

**7. What encoding routine is being used to obfuscate the domain name?**

The domain name is XORed with the string *1qaz2wsx3edc*.

**8. What is the significance of the CreateProcessA call at 0x0040106E?**

Function *main* calls subroutine at 0x401000 which calls CreateProcess function.

If *connect* call fails, subroutine at 0x401000 is called which creates a command line process which is binded with the socket descriptor to provide a reverse shell.

|               Main                |        Subroutine 0x401000        |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../9-2-1.PNG" >}} | {{< figure src="../9-2-2.PNG" >}} |


### Lab 3

**1. What DLLs are imported by Lab09-03.exe?**

The DLLs imported are KERNEL32.dll, NETAPI32.dll, DLL1.dll and DLL2.dll
With FLOSS we can observe that user32.dll and DLL3.dll are also imported.
{{< figure src="../9-3.PNG" >}}
{{< figure src="../9-3-1.PNG" >}}

**2. What is the base address requested by DLL1.dll, DLL2.dll, and DLL3.dll?**

The three DLLs requested the same address (0x1000000).

|             DLL1.dll              |             DLL2.dll              |             DLL3.dll              |
|:---------------------------------:|:---------------------------------:|:---------------------------------:|
| {{< figure src="../9-3-2.PNG" >}} | {{< figure src="../9-3-3.PNG" >}} | {{< figure src="../9-3-4.PNG" >}} |

**3. When you use OllyDbg to debug Lab09-03.exe, what is the assigned based address for: DLL1.dll, DLL2.dll, and DLL3.dll?**

We know from before that DLL3 is loaded dynamically, so we have to run the code until *LoadLibrary* is called with DLL3 as a parameter.
{{< figure src="../9-3-5.PNG" >}}

Once it has been loaded, we click `View ➡️ Memory` and observe that DLL2 and DLL3 have been relocated. In the following picture we can see where they have been loaded.
{{< figure src="../9-3-6.PNG" >}}
DLL2 is loaded at 0x30000 and DLL3 at 0x220000.

**4. When Lab09-03.exe calls an import function from DLL1.dll, what does this import function do?**

At address 0x401006, *DLL1Print* is called. The output that is printed at the command window after running that function is:
{{< figure src="../9-3-7.PNG" >}}

**5. When Lab09-03.exe calls WriteFile, what is the filename it writes to?**

As we can observe, *WriteFile* receives as a parameter a file handler, which is the return of the *DLL2ReturnJ* function. The next step is to analyze DLL2 file to gather more information about it.
{{< figure src="../9-3-8.PNG" >}}

This is what *DLL2ReturnJ* function does:
{{< figure src="../9-3-9.PNG" >}}

As this does not provide much information, the first thing that comes to my mind is to look for cross-references to that return value, which are listed below.
{{< figure src="../9-3-10.PNG" >}}

The last one is the one that we have already seen, so we need to check the other two references.
And *voilà*! The filename is *temp.txt*.
{{< figure src="../9-3-11.PNG" >}}

**6. When Lab09-03.exe creates a job using NetScheduleJobAdd, where does it get the data for the second parameter?**

*NetScheduleJobAdd* function receives three parameters. The second one refers to a buffer.
Line 24 shows that the buffer is heavily linked to the [*GetProcAddress*](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress) function, that retrieves the address of an exported function from the specified DLL, in this case DLL3.
{{< figure src="../9-3-12.PNG" >}}

**7. While running or debugging the program, you will see that it prints out three pieces of mystery data. What are the following: DLL 1 mystery data 1, DLL 2 mystery data 2, and DLL 3 mystery data 3?**

DLL 1 mystery data corresponds to the PID of the program:
{{< figure src="../9-3-14.PNG" >}}
{{< figure src="../9-3-15.PNG" >}}

We can also check this value by debugging the program and inspecting it with Process Explorer.
{{< figure src="../9-3-13.PNG" >}}

DLL 2 mystery data corresponds to the handle of *temp.txt* file.

|         DLL2Print function         | Return of CreateFile that is printed out |
|:----------------------------------:|:----------------------------------------:|
| {{< figure src="../9-3-16.PNG" >}} |    {{< figure src="../9-3-17.PNG" >}}    |

Finally, DLL 3 mistery data corresponds to the ping to "*www[.]malwareanalysis[.]com*" in memory.
{{< figure src="../9-3-18.PNG" >}}
{{< figure src="../9-3-19.PNG" >}}

**8. How can you load DLL2.dll into IDA Pro so that it matches the load address used by OllyDbg?**

We have to select Manual Load when loading the DLL with IDA Pro, and then type the new image base address (0x30000).