---
title: "Advanced Dynamic Techniques - WinDbg"
date: 2023-03-24T19:53:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, advanced, dynamic, analysis, x86, x64, assembly, registers, windbg, ghidra]
---
## Table of Contents
1. [Notes](#notes)
   1. [Drivers and Kernel Code](#drivers-and-kernel-code)
   2. [Setting Up Kernel Debugging](#setting-up-kernel-debugging)
      1. [Windows XP and x86 architecture](#windows-xp-and-x86-architecture)
      2. [Windows Vista, Windows 7, and x64 Versions](#windows-vista-windows-7-and-x64-versions)
   3. [Using WinDbg](#using-windbg)
   4. [Microsoft Symbols](#microsoft-symbols)
   5. [Kernel Debugging in Practice](#kernel-debugging-in-practice)
      1. [User-space code](#user-space-code)
      2. [Kernel-space code](#kernel-space-code)
   6. [Loading Drivers](#loading-drivers)
2. [Labs](#labs)
   1. [Lab 1](#lab-1)
   2. [Lab 2](#lab-2)
   3. [Lab 3](#lab-3)

## Notes
### Drivers and Kernel Code
Drivers must be loaded into the kernel, just as DLLs are loaded into processes. When a driver is first loaded, its ``DriverEntry`` procedure is called, similar to DLLMain for DLLs. 
Unlike DLLs, which expose functionality through the export table, drivers must register the address for callback functions, which will be called when a user-space software component requests a service. The registration happens in the ``DriverEntry`` routine. Windows creates a ``driver object`` structure, which is passed to the ``DriverEntry`` routine. The ``DriverEntry`` routine is responsible for filling this structure in with its callback functions. The ``DriverEntry`` routine then creates a device that can be accessed from user space, and the user-space application interacts with the driver by sending requests to that device.

The most commonly encountered request for a malicious kernel component is ``DeviceIoControl``, which is a generic request from a user-space module to a device managed by a driver. The user-space program passes an arbitrary length buffer of data as input and receives an arbitrary length buffer of data as output.

⚠️. **Some kernel-mode malware has no significant user-mode component. It creates no device object, and the kernel-mode driver executes on its own.**

Malicious drivers generally do not usually control hardware; instead, they interact with the main Windows kernel components, ``ntoskrnl.exe`` and ``hal.dll``. The ``ntoskrnl.exe`` component has the code for the core OS functions, and ``hal.dll`` has the code for interacting with the main hardware components.

### Setting Up Kernel Debugging
#### Windows XP and x86 architecture
Before you start editing the boot.ini file, take a snapshot of your virtual machine. Then copy the last line of your boot.ini file and add another entry. The line should be the same except that you should add the options ``/debug /debugport=COM1 /baudrate=115200``.

At VirtualBox, we have to configure the following settings:
{{< figure src="../10-1-3.PNG" >}}

And of course download [WinDbg Preview](https://apps.microsoft.com/store/detail/windbg-preview/9PGJGD53TN86?hl=es-es&gl=es&rtc=1).
{{< figure src="../10-1-6.PNG" >}}

#### Windows Vista, Windows 7, and x64 Versions
One major change is that since Windows Vista, the boot.ini file is no longer used to determine which OS to boot. Vista and later versions of Windows use a program called BCDEdit to edit the boot configuration data.

PatchGuard, implemented in the x64 versions of Windows starting with Windows XP, prevents third-party code from modifying the kernel. This includes modifications to the kernel code itself, modifications to system service tables, modifications to the IDT, and other patching techniques.

If you attach a kernel debugger after booting up, PatchGuard will cause a system crash.

### Using WinDbg
- **da**. Reads from memory and displays it as ASCII text.
- **du**. Reads from memory and displays it as Unicode text.
- **dd**. Reads from memory and displays it as 32-bit double words.
- **dwo**. Dereferences a 32-bit pointer and see the value at that location, e.g. ``du dwo (esp+4)``
- **bp**. Sets basic breakpoints.

### Microsoft Symbols
The format for referring to a symbol in WinDbg is as follows: ``moduleName!symbolName``
- *ntoskrnl.exe* is a special case and the module name is **nt**, not *ntoskrnl*.

``bu newModule!exportedFunction`` will instruct WinDbg to set a breakpoint on exportedFunction
- When analyzing kernel modules, the command ``bu $iment(driverName)`` will set a breakpoint on the entry point of a driver.

The x command allows you to search for functions or symbols using wildcards.
``x nt!*CreateProcess*`` will display exported functions as well as internal functions that include the string CreateProcess.

``ln`` command, which will list the closest symbol for a given memory address.

``dt nt!_DRIVER_OBJECT ADDRESS`` provides the driver object structure at that address, and it is useful to gather more information about it.

### Kernel Debugging in Practice
#### User-space code
- ``CreateService`` function together with the parameter ``dwService`` containing ``0x01`` value, which indicates that this is a kernel driver.
- ``DeviceIoControl`` function to send data to the driver

#### Kernel-space code
- ``!drvobj NameOfTheSuspiciousDriver``
- Sometimes the driver object will have a different name or !drvobj will fail. As an alternative, you can browse the driver objects with the !object \Driver command. This command lists all the objects in the \Driver namespace.
- ``dt nt!_DRIVER_OBJECT ADDRESS``
- The ``major function table`` tells us what is executed when the malicious driver is called from user space. The table has different functions at each index. Each index represents a different type of request, and the indices are found in the file *wdm.h* and start with ``IRP_MJ_``.
- ``dd OBJECT_ADDRESS+OFFSET_MAJOR_FUNCTION+IRP_MJ_DEVICE_CONTROL_INDEX*4 L1`` command finds the function that will be called to handle the DeviceIoControl request.
- ``!devhandles`` command obtains a list of all user-space applications that have a handle to that device. This command iterates through every handle table for every process, which takes a long time.

### Loading Drivers
If you have a malicious driver, but no user-space application to install it, you can load the driver using a loader such as the *OSR Driver Loader* tool.


## Labs

### Lab 1

**1. Does this program make any direct changes to the registry? (Use procmon to check.)**

Before starting with dynamic analysis, we can observe the imports from the .exe file. They are listed In the following screenshot:
{{< figure src="../10-1.PNG" >}}

The exercise introduction tell us that in order for the program to work properly, the driver must be placed in the C:\Windows\System32 directory.
After placing the driver in the correct path and running the .exe file while capturing the events with *Procmon* we obtain the following capture:
{{< figure src="../10-1-2.PNG" >}}

As you can see, the only **direct** change made to the registry was the call that is pointed with the red arrow.

**2. The user-space program calls the ControlService function. Can you set a breakpoint with WinDbg to see what is executed in the kernel as a result of the call to ControlService?**

[ControlService function](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-controlservice) sends a control code to a service.

In the following code snippet we can observe that the program creates and start a service. The fifth parameter (*dwService type*) is 0x01. This value indicates that this is a kernel driver.
{{< figure src="../10-1-4.PNG" >}}

This function is located at 0x00401080 as it can be seen in the image below.
{{< figure src="../10-1-5.PNG" >}}

We can set a breakpoint with OllyDbg and then observe any changes in WinDbg.
{{< figure src="../10-1-7.PNG" >}}

For example, after running the .exe file in OllyDbg and stopping the execution, we break and run the following command in WinDbg: ``!drvobj Lab10-01``
{{< figure src="../10-1-8.PNG" >}}

This indicates that the Driver has been loaded into 0x8996EC08. If we search for more information at that location with `dt nt!_DRIVER_OBJECT 8996ec08`, we observe the following:
{{< figure src="../10-1-9.PNG" >}}

As we want to know what is happening before the driver is unloaded we set a breakpoint at 0xF7A77486 with the command `bp 0xF7A77486`. Then, we resume the file in OllyDbg and the breakpoint is inmediately hit. With ``x nt!*CreateRegistryKey*`` we will display exported functions as well as internal functions that include the string CreateRegistryKey.
{{< figure src="../10-1-10.png" >}}

With *ln* command we can browse to that memory address and then observe the different operations that are performed.
{{< figure src="../10-1-11.PNG" >}}

If we analyse further the driver in Ghidra we can see that it is modifying registry keys related to Windows Firewall, particularly setting its value to 0.
{{< figure src="../10-1-12.PNG" >}}
{{< figure src="../10-1-13.PNG" >}}

**3. What does this program do?**

The program loads a driver that disables Windows Firewall.


### Lab 2

**1. Does this program create any files? If so, what are they?**

First of all we have to look at the imports and strings with FLOSS. This is the information that we have retrieved with basic static analysis:

|             Imports              |              Strings               |
|:--------------------------------:|:----------------------------------:|
| {{< figure src="../10-2.PNG" >}} | {{< figure src="../10-2-1.PNG" >}} |

If we run this sample together with Procmon we can observe that it creates a driver filled called "*Mlwx486.sys*"
{{< figure src="../10-2-2.PNG" >}}

This information can also be obtained by loading the file into Ghidra and looking at the main function. The file "*Lab10-02.exe*" has another file attached to itself in the resource section. It creates the driver file "*Mlwx486.sys*" and loads the resource into that file.
{{< figure src="../10-2-3.PNG" >}}

**2. Does this program have a kernel component?**

The program creates and start a service. The fifth parameter (*dwService type*) is 0x01. This value indicates that this is a kernel driver. 

**3. What does this program do?**

The file "*Lab10-02.exe*" has another file attached to itself in the resource section. It creates the driver file "*Mlwx486.sys*" and loads the resource into that file.

With ResourceHacker we are able to save that file and analyze it to obtain the functionality of the driver.
{{< figure src="../10-2-4.PNG" >}}

The first line in the picture above mentions the word **rootkit**. Hint?
Part of the chapter is dedicated to them, and it mentions the following:
``Although rootkits can employ a diverse array of techniques, in practice, one technique is used more than any other: System Service Descriptor Table (SSDT) hooking.``
It also says that if *NtQueryDirectoryFile*, among others functions, is hooked, it will change the value in the SSDT so that the rootkit code is called instead of the intended function in the kernel.

This can be confirmed by examining the driver file in Ghidra.
{{< figure src="../10-2-5.PNG" >}}


### Lab 3

**1. What does this program do?**

Firstly, we need to look at the imports and strings with FLOSS.

|             Imports              |              Strings               |
|:--------------------------------:|:----------------------------------:|
| {{< figure src="../10-3.PNG" >}} | {{< figure src="../10-3-1.PNG" >}} |

With dynamic analysis we have retrieved useful information. As you can observe in the following pictures, the malware sample queries the URL "www[.]malwareanalysisbook[.]com". 
{{< figure src="../10-3-4.PNG" >}}

If we take a look at this screenshot, we can see that requests to the malicious URL are made every 30 seconds.
{{< figure src="../10-3-3.PNG" >}}
{{< figure src="../10-3-5.PNG" >}}

On the other hand, with Procmon we have not acquired any indicator that shows registry or file creation and modification.

If we import the malware to Ghidra we can get more information about the sample. Lines 38 to 40 show an infinite loop that every 30 seconds queries the URL, as we mentioned before.
At the beginning of the main function, we can see that a service is created and started. Next, a file is created, although this operation is not captured by Procmon.
{{< figure src="../10-3-2.PNG" >}}

Regarding the .sys file, the following images shows its strings and entry function:

|              Strings               |           Entry function           |
|:----------------------------------:|:----------------------------------:|
| {{< figure src="../10-3-6.PNG" >}} | {{< figure src="../10-3-7.PNG" >}} |

**2. Once this program is running, how do you stop it?**

We can either reboot the machine or exit the Internet Explorer process.

**3. What does the kernel component do?**

Recalling basic dynamic analysis, the process did not appear in the Process List. This is achieved thanks to the driver, as it hides the process by unlinking it from the user processes.