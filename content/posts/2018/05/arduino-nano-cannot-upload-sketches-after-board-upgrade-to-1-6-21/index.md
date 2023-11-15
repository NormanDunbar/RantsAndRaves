---
title: "Arduino Nano - Cannot Upload Sketches after Board Upgrade to 1.6.21"
date: "2018-05-28"
categories: 
  - "gadgets-gizmos"
---

My Arduino Nano started to refuse to upload sketches after I upgraded the boards library to version 1.6.21 from 1.6.20. All I got was this:

```
...
avrdude: stk500\_recv(): programmer is not responding
avrdude: stk500\_getsync() attempt 1 of 10: not in sync: resp=0x00
avrdude: stk500\_recv(): programmer is not responding
avrdude: stk500\_getsync() attempt 2 of 10: not in sync: resp=0x00
...
```

Repeat 100 times!

After reverting back to the previous boards library, version 1.6.20, everything worked. I upgraded and everything stopped working.

The solution, from the arcduino.cc forums, is simple:

**Tools -> Processor -> ATmega328P (Old Bootloader)**

Read the forum thread at [https://forum.arduino.cc/index.php?topic=532983.0](https://forum.arduino.cc/index.php?topic=532983.0) if you wish. I'm just noting this here for my own reference and definitely not claiming any credit.
