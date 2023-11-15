---
title: "VirtualBox - Sending Special Key Combinations to the Guest OS."
date: "2014-03-02"
categories: 
  - "linux"
  - "virtualbox"
---

As standard, VirtualBox has a menu option (machine->Insert CTRL+ALT+Backspace or Insert CTRL+ALT+DEL) which is fine if you need these key combinations sending to the guest and not grabbed by the host, however, how can you send CTRL+ALT+F1 through CTRL+ALT+F8 to a Linux guest OS to get it to startup one of its virtual consoles?

Normally, those keys would be grabbed and actioned by the host OS rather than being passed to the guest. What to do?

Simple, since very early versions of VirtualBox (around 1.3.7 or 1.3.8) the Host (Virtual) key is a good substitute for CTRL+ALT and can be used for other special key combinations. On my Linux host OS, the host key is the right side CTRL key.

By pressing that, holding it down and pressing BackSpace, for example, I can reset the X system in the guest (Linux) OS.

The F1 through F8 keys work in a similar way if you press them after the Host key. In fact, Oracle have set VirtualBox up in such a way as to allow you to press any of the function keys from F1 through F12 and with the host key, will be passed as CTRL+ALT+Fn to the guest.
