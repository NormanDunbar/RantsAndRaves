---
title: "Linux Command to Retrieve Hardware Serial Numbers etc"
date: "2013-03-18"
categories: 
  - "linux"
---

Ever needed to obtain the serial number (or other details) for a remote server? Couldn't be bothered to walk/run/drive/fly all the way there just to read a sticky label on the back or bottom of said server? Read on then.

The command you want to run, as root, is `dmidecode`. For example, to get the make and model and serial number of a server, do this:

```bash
dmidecode -t system
```
The result will be similar to:

```text
# dmidecode 2.11
SMBIOS 2.5 present.

Handle 0x0002, DMI type 1, 27 bytes
System Information
        Manufacturer: Dell Inc.
        Product Name: Vostro 1720
        Version: Null
        Serial Number: 996C4L1
        UUID: Not Settable
        Wake-up Type: Power Switch
        SKU Number: Null
        Family: Vostro

Handle 0x000F, DMI type 12, 5 bytes
System Configuration Options
        Option 1: Jumper settings can be described here.

Handle 0x0018, DMI type 32, 20 bytes
System Boot Information
        Status: No errors detected
```

Other options for the `-t` parameter are:

- `bios` - tells you all about your bios.
- `system` - tells you about the system hardware.
- `baseboard` - all about the mother board.
- `chassis` - all you need to know about the "box" the system is made up of.
- `processor` - fairly obvious.
- `memory` - again, fairly obvious.
- `cache` - information about your CPU cache.
- `connector` - what sockets are present on the computer. USB, firewire, ethernet etc.
- `slot` - appears to be the bus information, and voltages present, supplied etc.

There's brief help available:

```bash
dmidecode --help
```
```text
Usage: dmidecode \[OPTIONS\]
Options are:
 -d, --dev-mem FILE     Read memory from device FILE (default: /dev/mem)
 -h, --help             Display this help text and exit
 -q, --quiet            Less verbose output
 -s, --string KEYWORD   Only display the value of the given DMI string
 -t, --type TYPE        Only display the entries of given type
 -u, --dump             Do not decode the entries
     --dump-bin FILE    Dump the DMI data to a binary file
     --from-dump FILE   Read the DMI data from a binary file
 -V, --version          Display the version and exit
```

However, to find out the different types you can supply, you need to supply an erroneous type:

```bash
dmidecode -t left_leg
```
```text
Invalid type keyword: left_leg
Valid type keywords are:
  bios
  system
  baseboard
  chassis
  processor
  memory
  cache
  connector
  slot
```

I've just used the command to obtain information about a server located 150 odd miles away from my comfy chair, running in an unattended site. That saved me a bit of time!

Have fun.
