---
title: "Advanced Static Techniques - x86 Disassembly"
date: 2022-12-19T09:50:58+05:30
description: ""
tags: [practical malware analysis, malware, reversing, advanced, static, analysis, x86, x64, assembly, registers]
---
## Table of Contents
1. [Opcodes and Endianness](#opcodes-and-endianness)
2. [Registers](#registers)
   1. [General registers](#general-registers)
   2. [Simple instructions](#simple-instructions)
   3. [The Stack](#the-stack)

This blog post collects some notes that I took while reading PMA Chapter 4. As it was not the first time dealing with assembly most of the concepts rang a bell.

## Opcodes and Endianness
Network data uses big-endian and an x86 program uses little-endian. 
Therefore, the IP address 127.0.0.1 will be represented as 0x7F000001 in big endian
format (over the network) and 0x0100007F in little-endian format (locally in memory)

## Registers
### General registers
Some x86 instructions use specific registers by definition.
- Multiplication and division instructions always use EAX and EDX.
- EAX generally contains the return value for function calls.
- EIP, the Instruction Pointer or program counter, is a register that contains the memory address of the next instruction to be executed for a program. 

### Simple instructions
- The *lea* instruction is used to put a memory address into the destination. For example, `lea eax, [ebx+8]` will put EBX+8 into EAX. In contrast, `mov eax, [ebx+8]` loads the data at the memory address specified by EBX+8. Therefore, `lea eax, [ebx+8]` would be the same as `mov eax, ebx+8`; however, a *mov* instruction like that is invalid. The lea instruction is not used exclusively to refer to memory addresses. It is useful when calculating values, because it requires fewer instructions.
- Regarding *mul* instruction, the result is stored as a 64-bit value across two registers: EDX and EAX. EDX stores the most significant 32 bits of the operations, and EAX stores the least significant 32 bits.
- The *div*  instruction does the same thing as *mul*, except in the opposite direction: It divides the 64 bits across EDX and EAX by value. Therefore, the EDX and EAX registers must be set up appropriately before the division occurs. The result of the division operation is stored in EAX, and the remainder is stored in EDX.
- During malware analysis, if you encounter a function containing only the instructions *xor*, *or*, *and*, *shl*, *ror*, *shr*, or *rol* repeatedly and seemingly randomly, you have probably encountered an **encryption** or **compression function**.

### The Stack
x86 architecture provides additional instructions for popping and pushing, the most popular of which are *pusha* and *pushad*. These instructions push all the registers onto the stack and are commonly used with *popa* and *popad*, which pop all the registers off the stack. The *pusha* and *pushad* functions operate as follows:
- *pusha* pushes the 16-bit registers on the stack in the following order: AX, CX, DX, BX, SP, BP, SI, DI.
- *pushad* pushes the 32-bit registers on the stack in the following order: EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI.
These instructions are typically encountered in shellcode when someone wants to save the current state of the registers to the stack so that they can be restored at a later time. Compilers rarely use these instructions, so seeing them often indicates someone **hand-coded assembly and/or shellcode**.

