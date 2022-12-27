---
title: "Advanced Dynamic Techniques - Debugging"
date: 2022-12-27T12:25:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, advanced, dynamic, analysis, x86, x64, assembly, registers, ollydbg, windbg, breakpoints, kernel]
---

## Notes

### Kernel vs. User-Mode Debugging
It is more challenging to debug kernel-mode code than to debug user-mode code because you usually need two different systems for kernel mode.

Kernel debugging is performed on two systems because there is only one kernel; if the kernel is at a breakpoint, no applications can be running on the system. One system runs the code that is being debugged, and another runs the debugger. Additionally, the OS must be configured to allow for kernel debugging, and you must connect the two machines.

- WinDbg is currently the only popular tool that supports kernel debugging.

### Breakpoints
#### Software breakpoints
Most popular debuggers set a software execution breakpoint by default. The debugger implements a software breakpoint by overwriting the first byte of an instruction with 0xCC, the instruction for INT 3, the breakpoint interrupt designed for use with debuggers. When the 0xCC instruction is executed, the OS generates an exception and transfers control to the debugger.

#### Hardware breakpoints
Every time the processor executes an instruction, there is hardware to detect if the instruction pointer is equal to the breakpoint address. Unlike software breakpoints, with hardware breakpoints, it doesn’t matter which bytes are stored at that location.

Unfortunately, hardware execution breakpoints have one major drawback: only four hardware registers store breakpoint addresses.

One further drawback of hardware breakpoints is that they are easy to modify by the running program. There are eight debug registers in the chipset, but only six are used. The first four, DR0 through DR3, store the address of a breakpoint. The debug control register (DR7) stores information on whether the values in DR0 through DR3 are enabled and whether they represent
read, write, or execution breakpoints. Malicious programs can modify these registers, often to interfere with debuggers. Thankfully, x86 chips have a feature to protect against this. By setting the General Detect flag in the DR7 register, you will trigger a breakpoint to occur prior to executing any ``mov`` instruction that is accessing a debug register.

### Exceptions
#### First- and Second-Chance Exceptions
When analyzing malware, you are generally not looking for bugs, so first chance exceptions can often be ignored. Malware may intentionally generate first-chance exceptions in order to make the program difficult to debug.

Second-chance exceptions cannot be ignored, because the program cannot continue running. If you encounter second-chance exceptions while debugging malware, there may be bugs in the malware that are causing it to crash, but it is more likely that the malware doesn’t like the environment in which it is running.