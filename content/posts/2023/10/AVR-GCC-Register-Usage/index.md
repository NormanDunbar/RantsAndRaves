---
title: "AVR-GCC Register Usage"
date: 2023-10-23T15:39:07+01:00
lastmod: 2023-10-23T15:39:07+01:00
description: "Which registers do I need to preserve when calling C++ code from assembly, and vice versa?"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocFolding: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
categories:
    - "Assembly Language"
    - "Avr-as"
    - "Avr-gcc"
---

When writing AVR assembly code, there are protocols that must be followed when using and abusing the various registers available in the ATmega328P device, which is what I'm using, built in to many Arduino boards.

The assembly language written can be compiled in the Arduino IDE using the GCC/G++ compiler controller and this, in turn, will call the avr-as assembler to do the actual work. All you need to do is add a new file to your Arduino sketch, and name it `*.S` where the name can be anything, but the extension *must* be an upper case 'S'.

### Special Registers

#### R0: Temporary Register

`R0` is the temporary register used by the C++ compiler generated code. To this end, it *must* be preserved by the assembly code if it is used there. Obviously, it should also be restored before calling or returning to C++.


#### R1: Zero Register

`R1` is the zero register. The code generated by the C++ compiler expects the register toalways hold the value zero. If your assembly code uses this register for any reason, it *must* be cleared back to zero before calling or returning to C++.



### Call Saved Registers

`R2`-`R17` plus `R28` and `R29` are the *Call Saved Registers*. What this means is that these *must* be saved if they are used in your assembly code, and restored before calling or returning to C++.


### Call Clobbered Registers

`R18`-`R27` plus `R30` and `R31` are the *Call Used/Clobbered Registers*. What this means is that these can be used freely in your assembly code, however, they are clobbered by the C++ compiler's generated code, so if you call a C++ function, you *must* preserve them if you have values in them which you will use on return from the C++ call.

### Summary

| Register | Category | Special Considerations |
| :--- | :--- | :--- |
| R0 | Temporary Register | Must be preserved if used by assembly code, and restored before calling or returning to C++ code. |
| R1 | Zero Register | Must be cleared to zero before calling or returning to C++. |
| R2 to R17 | Call Saved Registers | If used in assembly code, must have their values preserved before use, and restored before calling or returning to C++. |
| R18 to R27 | Call Clobbered Registers | If used in assembly code, must have their values preserved before calling C++ functions, and restored on return. |
| R28 and R29 | Call Saved Registers | If used in assembly code, must have their values preserved before use, and restored before calling or returning to C++. |
| R30 and R31 | Call Clobbered Registers | If used in assembly code, must have their values preserved before calling C++ functions, and restored on return. |