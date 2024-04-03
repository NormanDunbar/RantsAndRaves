---
title: "Configuring the Arduino IDE to Allow AVR Debugging"
date: 2024-04-03T09:54:00+01:00
lastmod: 2024-04-03T09:54:00+01:00
description: "Configuring the Arduino IDE to Allow AVR Debugging"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocFolding: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
categories:
    - "Simulavr"
    - "Debugging"
    - "Arduino"
    - "AVR"
---

In a [previous post](/posts/2023/10/debugging-avr-code-with-simulavr) I mentioned that the Arduino IDE needed to be setup in order to allow the Sketch->Optimize for debugging menu option to have any effect. The IDE "knows" that ATmega328P microcontrollers, as used on the Uno R3, Duemilanove, Nano and so on, do not have actual hardware debugging, so the option has no effect at all.

However, with a little sneaky configuration, we can change things so that we can optimise our code for debugging with *avr-gdb* and *simulAVR* as described [here](/posts/2023/10/debugging-avr-code-with-simulavr).

Close the IDE and navigate to `$HOME/.arduino15/packages/arduino/hardware/avr/1.8.6` and open the file named `platform.local.txt` in your favourite text editor. If there is no file of that name there, create the file---the name is all lower case---and open it.

Add the text in the Listing below to the file. Comments are prefixed by the '#' character and if you wish, just ignore them. All non-comment lines are a single line each---they have wrapped around here to fit the page.

```
## See https://github.com/arduino/ArduinoCore-avr/issues/554

# Fallback property definition for compatibility with 
# development tools that don't have the "Optimize for 
# Debugging" control.

compiler.optimization_flags=-Os -g

# See: https://arduino.github.io/arduino-cli/latest/platform-
# specification/#optimization-level-for-debugging.

compiler.optimization_flags.release=-Os -g

compiler.optimization_flags.debug=-O0 -g

compiler.c.flags=-c {compiler.optimization_flags} 
{compiler.warning_flags} -std=gnu11 -ffunction-sections 
-fdata-sections -MMD -flto -fno-fat-lto-objects

compiler.c.elf.flags={compiler.optimization_flags} 
{compiler.warning_flags} -flto -fuse-linker-plugin 
-Wl,--gc-sections

compiler.cpp.flags=-c {compiler.warning_flags} 
{compiler.optimization_flags} -std=gnu++11 -fpermissive 
-fno-exceptions -ffunction-sections -fdata-sections 
-fno-threadsafe-statics -Wno-error=narrowing -MMD -flto
```

**NOTE**: By creating (or using) `platform.local.txt` you will avoid having to redo all this configuration when you next update the IDE to a newer version. Had we simply added the code to `platform.txt` our changes would have been overwritten on every update.

Save the `platform.local.txt` file and open the IDE again; open a sketch you have compiled previously; click Sketch->Optimize for Debugging; then compile the sketch in verbose mode. You should see that the compiler and linker options  now display something similar to the following:

```
Compiling sketch...
.../avr-gcc/7.3.0-atmel3.6.1-arduino7/bin/avr-g++ -c -O0 -g ... 

...

Linking everything together...
.../avr-gcc/7.3.0-atmel3.6.1-arduino7/bin/avr-gcc -O0 -g ...
```

**NOTE**: You can ignore the warning messages that advise about `#warning "Compiler optimizations disabled; functions from <util/delay.h> won't work as designed` as you will not be uploading the debug version of the sketch to your board.
    
From now on, always untick the `Optimize for Debugging` option in the IDE before compiling and uploading to your board. The option is only required for debugging in the simulator.
    
Debugging creates much larger output files and it's possible that a sketch you have previously uploaded quite happily will no longer upload if you forget about this option. One of my sketches compiles to 10,570 bytes with debugging enabled but only 2,410 bytes without debugging.

