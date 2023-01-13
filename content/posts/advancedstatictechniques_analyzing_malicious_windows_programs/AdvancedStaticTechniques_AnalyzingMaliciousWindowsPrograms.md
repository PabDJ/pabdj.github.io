---
title: "Advanced Static Techniques - Analyzing Malicious Windows Programs"
date: 2022-12-26T14:25:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, advanced, static, analysis, x86, x64, assembly, registers, ida, ghidra]
---
## Table of Contents
1. [Notes](#notes)
   1. [Windows API types](#windows-api-types)
   2. [Windows Registry](#windows-registry)
      1. [Registry Root Keys](#registry-root-keys)
   3. [Networking APIs](#networking-apis)
   4. [Following running malware](#following-running-malware)
   5. [Kernel vs. User Mode](#kernel-vs-user-mode)
   6. [The Native API](#the-native-api)
2. [Labs](#labs)
   1. [Lab 1](#lab-1)
   2. [Lab 2](#lab-2)
   3. [Lab 3](#lab-3)

## Notes
- DWORD and WORD types represent 32-bit and 16-bit unsigned integers. Windows does not use int, short or unsigned.

### Windows API types
- **Handles (H)**. A reference to an object.
- **Long Pointer (LP)**. A pointer to another type.
- **CreateFile**. This function is used to create and open files. The parameter *dwCreationDisposition* controls whether the CreateFile function creates a new file or opens an existing one.
- **CreateFileMapping** and **MapViewOfFile**. File mappings are commonly used by malware writers because they allow a file to be loaded into memory and manipulated easily. The CreateFileMapping function loads a file from disk into memory. After obtaining a map of the file, the malware can parse the PE header and make all necessary changes to the file in memory, thereby causing the PE file to be executed as if it had been loaded by the OS loader.
- **Shared Files**. Shared files are special files with names that start with \\serverName\share or \\?\serverName\share. They access directories or files in a shared folder stored on a network.
- **Files Accessible via Namespaces**. Additional files are accessible via namespaces within the OS. It allows to directly access physical devices while ignoring its file system, thereby allowing it to modify the disk in ways that are not possible through the normal API. Using this method, the malware might be able to read and write data to an unallocated sector without creating or accessing files, which allows it to avoid detection by antivirus and security programs.
- **Alternate Data Streams**. It allows additional data to be added to an existing file within NTFS, essentially adding one file to another. The extra data does not show up in a directory listing, and it is not shown when displaying the contents of the file; it’s visible only when you access the stream.

### Windows Registry
#### Registry Root Keys
The registry is split into the following five root keys:
- **HKEY_LOCAL_MACHINE (HKLM)**.
- **HKEY_CURRENT_USER (HKCU)**.
- **HKEY_CLASSES_ROOT**.
- **HKEY_CURRENT_CONFIG**.
- **HKEY_USERS**.

#### Common Registry Functions
The following are the most common registry functions that malware uses:
- **RegOpenKeyEx**.
- **RegSetValueEx**.
- **RegGetValue**.

### Networking APIs
The ***WSAStartup*** function must be called before any other networking functions in order to allocate resources for the networking libraries. When looking for the start of network connections while debugging code, it is useful to **set a breakpoint on *WSAStartup***, because the start of networking should follow shortly.

The ***WinINet API*** implements protocols, such as HTTP and FTP, at the application layer.

### Following running malware
Malware can use ***CreateThread*** in multiple ways, such as the following:
- Malware can use ***CreateThread*** to load a new malicious library into a process, with ***CreateThread*** called and the address of ***LoadLibrary*** specified as the start address. (The argument passed to ***CreateThread*** is the name of the library to be loaded. The new DLL is loaded into memory in the process, and ***DllMain*** is called.)
- Malware can create two new threads for input and output: one to listen on a socket or pipe and then output that to standard input of a process, and the other to read from standard output and send that to a socket or pipe. The malware’s goal is to send all information to a single socket or pipe in order to communicate seamlessly with the running application.

In addition to threads, Microsoft systems use ***fibers***. Fibers are like threads, but are managed by a thread, rather than by the OS. Fibers share a single thread context.

Another way for malware to execute additional code is by installing it as a service. Windows allows tasks to run without their own processes or threads by using services that run as background applications; code is scheduled and run by the Windows service manager without user input.
There are several key functions to look for:
- **OpenSCManager**. 
- **CreateService**.
- **StartService**.

The Windows OS supports several service types, which execute in unique ways. The one most commonly used by malware is the ***WIN32_SHARE_PROCESS*** type, which stores the code for the service in a DLL, and combines several services in a single, shared process. The ***WIN32_OWN_PROCESS*** type and***KERNEL_DRIVER*** type are also used.
The *qc* command queries a service’s configuration options by accessing the same information as the registry entry in a more readable way.

``C:\Users\User1>sc qc "VMware NAT Service"``


Each thread that uses COM must call the ***OleInitialize*** or ***CoInitializeEx*** function at least once prior to calling any other COM library functions. The ***CoCreateInstance*** function is used to get access to COM functionality.
- One common function used by malware is ***Navigate***, which allows a program to launch Internet Explorer and access a web address.
In order to identify what a malicious program is doing when it calls a COM function, malware analysts must determine which offset a function is stored at, which can be tricky. IDA Pro stores the offsets and structures for common interfaces, which can be explored via the structure subview. Press the *INSERT* key to add a structure, and then click **Add Standard Structure**. The name of the structure to add is ***InterfaceNameVtbl***.
Malware that implements a COM server is usually easy to detect because it exports several functions, including ***DllCanUnloadNow***, ***DllGetClassObject***, ***DllInstall***, ***DllRegisterServer***, and ***DllUnregisterServer***, which all must be exported by COM servers.

*Structured Exception Handling* (SEH) is the Windows mechanism for handling exceptions. In 32-bit systems, SEH information is stored on the **stack**. The special location ***fs:0*** points to an address on the stack that stores the exception information.

### Kernel vs. User Mode
When you call a Windows API function that manipulates kernel structures, it will make a call into the kernel. The presence of the *SYSENTER*, *SYSCALL*, or *INT 0x2E* instruction in disassembly indicates that a call is being made into the kernel.

### The Native API
The Native API is a lower-level interface for interacting with Windows that is rarely used by non-malicious programs but is popular among malware writers.
User applications are given access to user APIs such as *kernel32.dll* and other DLLs, which call *ntdll.dll*, a special DLL that manages interactions between user space and the kernel. The processor then switches to kernel mode and executes a function in the kernel, normally located in *ntoskrnl.exe*.
There are a series of Native API calls that can be used to get information about the system, processes, threads, handles, and other items. These include ***NtQuerySystemInformation***, ***NtQueryInformationProcess***, ***NtQueryInformationThread***, ***NtQueryInformationFile***, ***NtQueryInformationKey*** and ***NtContinue***.


## Labs

### Lab 1

**1. How does this program ensure that it continues running (achieves persistence) when the computer is restarted?**

Before going that deep on how the malware achieves persistence we have to perform a basic static and dynamic analysis.

Regarding the imports we can observe that it calls *InternetOpenA* and *InternetOpenUrlA* for networking capabilities. It also imports some service functions such as *OpenSCManagerA* and *CreateServiceA*. Last but not least, it imports functions such as *CreateThread*, *CreateMutex*, *OpenMutex* and *GetStartupInfo* among others.

Inside the strings we see things like "*MalService*", "*hxxp://www[.]malwareanalysisbook[.]com*", or "*Internet Explorer 8.0*".

Now, let's get a better overview of the malware with a basic dynamic analysis.
With the following screenshots we have gathered a lot of information.

👉 Wireshark 
{{< figure src="../7-1.PNG" >}}

👉 Regshot
{{< figure src="../7-1-1.PNG" >}}

👉 ProcessExplorer
{{< figure src="../7-1-2.PNG" >}}

Looking at Regshot screenshot we observe that the sample adds new keys to the registry with the name "*Malservice*".

**2. Why does this program use a mutex?**

It checks if the victim machine is already infected by checking if it contains an entry to the malicious URL in the history.

**3. What is a good host-based signature to use for detecting this program?**

The service "*Malservice*" and the mutex "*HGL345*" are good  host-based signature.

**4. What is a good network-based signature for detecting this malware?**

The URL "*hxxp://www[.]malwareanalysisbook[.]com*" and the User-Agent "*Internet Explorer 8.0*".

**5. What is the purpose of this program?**

First of all we land into main.
{{< figure src="../7-1-3.PNG" >}}

From there, we jump into subroutine located at 0x401040.
{{< figure src="../7-1-4.PNG" >}}

A timer is set to start on January 1st, 2100 and 20 threads are created which perform the following actions.
{{< figure src="../7-1-5.PNG" >}}

An infinite loop is done in order to perform a great amount of requests to the URL. That means that a Denial of Service (DoS) attack is created.

**6. When will this program finish executing?**

The program will never end as the threads are continuously executing.


### Lab 2

**1. How does this program achieve persistence?**

The program calls *OleInitialize* function and after that it gets access to COM functionality with *CoCreateInstance* function.
{{< figure src="../7-2.PNG" >}}

In order to get more information we need to gather information related to the parameters that *CoCreateInstance* function receives. IDA displays the information more beautiful in comparison with Ghidra.

|              Ghidra               |           IDA Free v8.1           |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../7-2-1.PNG" >}} | {{< figure src="../7-2-2.PNG" >}} |

The CLSID value is "{0002DF01-0000-0000-C000-000000000046}" so the next step is to search for it in the Windows Registry.
{{< figure src="../7-2-3.PNG" >}}

The malware sample is making an instance of the Internet Explorer object and if we take a look at the disassembly previously shown, in line 22 we can it calls the object with an offset equals to 0x2C which looks familiar.
In this Chapter, it was mentioned that function 0x2C is *Navigate* function.

I tried to find this by myself, but I struggled through the process. Along the way I found this post from Mandiant.

- [Reversing Malware Command and Control: From Sockets to COM](https://www.mandiant.com/resources/blog/reversing-malware-command-control-sockets)

``This function is at offset 2Ch from the beginning of the ppv data structure, and you can tell that the binary is calling this function because the call instruction accesses this specific offset from the beginning of the structure ("call dword ptr [ecx+**2Ch**]"). There is no clear-cut way to go directly from the Microsoft documentation of this COM interface to seeing the actually internal offset used by each member function [...]``

Anyway, as we can observe, there is no sign of persistence in the code.

**2. What is the purpose of this program?**

It browses to a malicious URL "*hxxp://www[.]malwareanalysisbook[.]com/ad.html*".

**3. When will this program finish executing?**

It will finish after making the request to the malicious URL.

### Lab 3

**1. How does this program achieve persistence to ensure that it continues running when the computer is restarted?**

The first thing is running strings in order to have a first contact with the sample. This way we can observe the imports and possible IoCs.

|            .exe file            |             .dll file             |
|:-------------------------------:|:---------------------------------:|
| {{< figure src="../7-3.PNG" >}} | {{< figure src="../7-3-1.PNG" >}} |

The .exe file deals with file management functions such as *CreateFile*, *CopyFile* or *FindNextFile*. An important feature of this file is that it plays with the similarity of number "1" and lowercase letter "L". It might be used for stealthiness.

Regarding the .dll file it works with mutex functions and a suspicious IP address appears "127[.]26[.]152[.]13".

Unfortunately, when performing a basic dynamic analysis nothing valuable is found. The last resource we have is disassembling the file with Ghidra.

{{< figure src="../7-3-2.PNG" >}}

From the screenshot above we can observe that the program calls *CreateFileMapping* function that loads a file from disk into memory. The *MapViewOfFile* function returns a pointer to the base  address of the mapping, which can be used to access the file in memory. The program calling these functions can use the pointer returned from *MapViewOfFile* to read and write anywhere in the file.

|      First part of the code       |          End of the code          |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../7-3-3.PNG" >}} | {{< figure src="../7-3-4.PNG" >}} |

The way of achieving persistence is by substituting one file by another with similar names.

**2. What are two good host-based signatures for this malware?**

File "*kerne132.dll*" and mutex "*SADFHUHF*".

{{< figure src="../7-3-5.PNG" >}}

**3. What is the purpose of this program?**

The .exe file installs the .dll file in the system, which is used for connecting to an IP address and establishing a remote connection with a server.

{{< figure src="../7-3-6.PNG" >}}

|          *hello* command          |    *sleep* and *exec* commands    |
|:---------------------------------:|:---------------------------------:|
| {{< figure src="../7-3-7.PNG" >}} | {{< figure src="../7-3-8.PNG" >}} |

In order to end the connection there is a q(uit) command.

**4. How could you remove this malware once it is installed?**

Something that I missed while analyzing the sample is that it infects **all** .exe files in the system. In our case the best option to remove the malware is by restoring the snapshot that we have. In a non-lab machine we would have to restore kernel32.dll in the place of *kernel132.dll*.