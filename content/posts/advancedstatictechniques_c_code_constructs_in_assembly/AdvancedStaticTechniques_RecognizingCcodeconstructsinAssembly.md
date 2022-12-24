---
title: "Advanced Static Techniques - Recognizing C code constructs in Assembly"
date: 2022-12-24T14:15:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, advanced, static, analysis, x86, x64, assembly, registers, ida, ghidra]
---
## Notes
### Global vs. Local Variables
- Global variables are referenced by memory addresses, and the local variables are referenced by the stack addresses.

### Understanding Function Call Conventions
- *cdecl* is one of the most popular conventions. In *cdecl*, parameters are pushed onto the stack from right to left, the caller cleans up the stack when the function is complete, and the return value is stored in EAX.
- The popular *stdcall* convention is similar to *cdecl*, except *stdcall* requires the callee to clean up the stack when the function is complete.
- *stdcall* is the standard calling convention for the Windows API. Any code calling these API functions will not need to clean up the stack, since that’s the responsibility of the DLLs that implement the code for the API function.
- In *fastcall*, the first few arguments (typically two) are passed in registers, with the most commonly used registers being EDX and ECX (the Microsoft fastcall convention). Additional arguments are loaded from right to left, and the calling function is usually responsible for cleaning up the stack, if necessary.
- Microsoft Visual Studio **pushes** the parameters onto the stack and GNU Compiler Collection (GCC) **moves** the parameters onto the stack before the call.


## Labs
This note is being written after struggling with Lab 1 and 2. 
Long story short, I was getting confused with **main** vs **entry point** because neither Ghidra nor IDA Free v7.0 were landing on main. In fact, they were doing it on entry point but if we mix that I am still a noob and function *main* was not labelled we can end up in a rabbit hole.
As I am not an expert, I will leave this two YouTube videos that help me out with this issue.

{{< youtube suwZB3EA_u4 >}}


Taking into account the trick that [@herrcore](https://twitter.com/herrcore) mentions in his video we have to search for three consecutive PUSH/MOV instructions, depending on whether we are working with x86 arch or x64.

|               Ghidra               |            IDA Free v8.1            |
|:----------------------------------:|:-----------------------------------:|
| {{< figure src="../Ghidra.PNG" >}} | {{< figure src="../IDAFree.PNG" >}} |

⚠️ **DON'T PANIC!** IDA Free v8.1 works fine and detects *main* function without any problem, but I will always recommend to understand how things work instead of relying on any tool 😉


### Lab 1

**1. What is the major code construct found in the only subroutine called by main?**

The subroutine is called at 0x401000. If we take a look at the function graph we can observe that it is an if statement.

**2. What is the subroutine located at 0x40105F?**

I have to admit that I fell in a rabbit hole with this question. When I gave up because I did not have any clue, I looked at the solutions and the answer was "*printf*". 

My mistake was to try to understand what the actual subroutine was doing instead of observing in what context it was called.
Below you can observe that in both screenshots my approach was to learn how the function worked.

|            IDA Free             |              Ghidra               |
|:-------------------------------:|:---------------------------------:|
| {{< figure src="../6-1.PNG" >}} | {{< figure src="../6-1-1.PNG" >}} |

If you take a look at the cross-references of the function, you can observe that it is being used as a printf function.
{{< figure src="../6-1-2.PNG" >}}

{{< figure src="../6-1-3.PNG" >}}

**3. What is the purpose of this program?**

After taking a look at the strings, we can observe some suspicious ones like *Error 1.1. No Internet* and *Success: Internet Connection*. Apart from that, if we analyse the imports, we can think about some malicious capabilities. Some of them are: *VirtualAlloc*, *GetCurrentProcess*, *GetStartupInfoA*, etc.

With basic dynamic techniques we cannot observe anything clear, so we have to jump into disassembly.

In subroutine 0x401000, we see that [InternetGetConnectedState](https://learn.microsoft.com/en-us/windows/win32/api/wininet/nf-wininet-internetgetconnectedstate) function is called. The program tries to receive information about the machine, in this case if it has network connection to the Internet.


### Lab 2

**1. What operation does the first subroutine called by main perform?**

IDA Free v8.1 directly lands on main but Ghidra, again, doesn't. In order to find main, as I mentioned before we have to find three consecutive PUSH instructions that involve EAX.

{{< figure src="../6-2.PNG" >}}

Once we have located main, we rename it, jump into it and find the subroutine.

The first subroutine in *main* is located at 0x401000, and it is an if statement.

**2. What is the subroutine located at 0x40117F?**

This function is the same one as Lab 1.2 answer, a printf function.

**3. What does the second subroutine called by main do?**

The second subroutine that is called is at 0x00401040. It tries to connect "*hxxp://www.practicalmalwareanalysis.com/cc.htm*" and tries to read a file in order to get a command.

The pictures below show the function graph.
{{< figure src="../6-2-1.PNG" >}}
{{< figure src="../6-2-2.PNG" >}}

**4. What type of code construct is used in this subroutine?**

It also uses an if statement.

**5. Are there any network-based indicators for this program?**

The main network-based indicator is the URL "*hxxp://www.practicalmalwareanalysis.com/cc.htm*"

**6. What is the purpose of this malware?**

This lab is a continuation of the sample Lab06-01.exe which checks if the machine has internet connection and if this call returns a successful confirmation, it tries to establish a connection with the URL mentioned before and gets some commands from there.


### Lab 3

**1. Compare the calls in main to Lab 6-2’s main method. What is the new function called from main?**

The new function is located at 0x401130.

**2. What parameters does this new function take?**

It takes two parameters. The first one is the return of subroutine at 0x401040 and if we observe that in the following line it is compared again the string '\0' and it prints out information related to a command, we can think that this parameter is linked to a command that the code receives from somewhere else.
Regarding the second parameter, it is the same one as the *main* function receives. Do you remember the classical *int main (int argc, char *argv)*? Here it is!

{{< figure src="../6-2-3.PNG" >}}

**3. What major code construct does this function contain?**

It contains a switch statement.
{{< figure src="../6-2-4.PNG" >}}

**4. What can this function do?**

It creates a directory which contains a file. In order to achieve persistence it copies itself into the registry and afterwards it deletes the file as a form of evasion.

**5. Are there any host-based indicators for this malware?**

There are some hardcoded paths like the ones shown in the image.
{{< figure src="../6-2-5.PNG" >}}

**6. What is the purpose of this malware?**

This malware checks if the machine has internet connection and if this call returns a successful confirmation, it tries to establish a connection with the URL mentioned before and gets some commands from there. Once they have been retrieved from the URL, they are used in order to create a directory which contains a file. For achieving persistence it copies itself into the registry and afterwards it deletes the file as a form of evasion.


### Lab 4

**1. What is the difference between the calls made from the main method in Labs 3 and 4?**

There is not a big difference between the two labs regarding the calls inside main.

|              Lab 3              |               Lab 4               |
|:-------------------------------:|:---------------------------------:|
| {{< figure src="../6-3.PNG" >}} | {{< figure src="../6-3-1.PNG" >}} |

**2. What new code construct has been added to main?**

A while loop has been added to main.

**3. What is the difference between this lab’s parse HTML function and those of the previous labs?**

This function calls subroutine located at 0x4012E6 because Internet Agent is slightly different.

**4. How long will this program run? (Assume that it is connected to the Internet.)**

It will last for 1440 minutes which is the same as 24 hours. There are 1440 executions in the loop and each one is performed in 60 seconds.

{{< figure src="../6-3-3.PNG" >}}

**5. Are there any new network-based indicators for this malware?**

The User-Agent with Internet Explorer.
{{< figure src="../6-3-4.PNG" >}}

**6. What is the purpose of this malware?**

This malware checks if the machine has internet connection and if this call returns a successful confirmation, it tries to establish a connection with the URL mentioned before with a unique User-Agent and gets some commands from there. Once they have been retrieved from the URL, they are used in order to create a directory which contains a file. For achieving persistence it copies itself into the registry and afterwards it deletes the file as a form of evasion. The program's execution is 24 hours.