---
title: "NOOBS For Raspberry Pi"
date: "2013-06-04"
categories: 
  - "linux"
  - "raspberry-pi"
---

**Updated 11th January 2015 to document NOOBS 1.3.11.**

NOOBS is the latest user friendly installation system from the Raspberry Pi. It allows you the ability to choose one of 7 (currently) Operating Systems to run on your Pi and a separate data partition to save your possibly shared data. You can pick and choose and change your OS at any time you wish simply by rebooting and holding the SHIFT key down. However, any user data on the SD card will be lost each time.

NOOBS is "New Out Of Box Software" by the way. Want to know more? Read on...

There are plenty of places on the web that tell you how to use NOOBS to set up a system, but so far, they all assume one thing, that you are using some flavour of Windows. I'm not.

One fine example is this video on You Tube - [http://www.youtube.com/watch?v=TyFDaMpdh2c](http://www.youtube.com/watch?v=TyFDaMpdh2c "http://www.youtube.com/watch?v=TyFDaMpdh2c")

So, what follows is a quick and easy guide for the Linux user to get a new SD card ready for use in their Raspberry Pi.

## Initialise Your SD Card

The SD card needs to be formatted as FAT32 initially. Most cards come with this formatting, but if you have stolen a card from a camera or similar device, it might be best to properly initialise it. Just in case.

You will need a 4GB or larger card. NOOBS adds a recovery partition to the card which uses up some space, and holds the various distros and support files for the "recovery" process. Basically, the installer lives on the card.

**The following assumes you have root privileges, so either `su -` or prefix all commands, unless otherwise advised, with `sudo`** If you see commands prefixed by a '**#**' prompt, that means that the command should be run with sudo or as root. If the commands have a '**$**' prompt, then you can run those commands as your normal user.

My laptop comes with an SD card slot, which is always device `/dev/mmcblk0` and partitions on the SD card are named as `/dev/mmcblk0pn` where the 'n' part starts at 1 and increases as desired.

First we need to find the device, assuming your system is different to mine. The easiest way is to run the following as root:

```bash
fdisk -l
```

The parameter is a lower case L for "list". This command lists all the mounted and unmounted devices on your computer. Make sure you don't have the SD card in the slot yet though!

Now, insert the card, and run the command again. If you are prompted to mount the card or open it in a file manager etc, ignore it. We don't want or need the card to be mounted.

{{< alert theme="info">}}
As mentioned, some Linux distros may auto-mount the SD card when you put it in the slot. If yours does, then unmount it or "safely remove" it - whichever option you are given.
 
On my Linux Mint laptop, under KDE, I get prompted with a pop-up telling me what partitions are on the newly inserted card, and offering me the choice to open in Dolphin or just to mount the partition. If I ignore the pop-up for 5 seconds or so, it vanishes, and the card remains unmounted.
 
If you use KDE and Dolphin, and if the card is auto-mounted, open Dolphin, find the SD card on the list of devices down the left side, right click it and the top option is to "safely remove" the device.
 
You might need to "safely remove" all the partitions on the SD Card, not just one, if all of them have been auto-mounted.
{{< /alert >}}

Compare the two listings and you should see something like the following:

```text
...
Disk /dev/mmcblk0: 7948 MB, 7948206080 bytes
4 heads, 16 sectors/track, 242560 cylinders, total 15523840 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000dbfc6

        Device Boot      Start         End      Blocks   Id  System
/dev/mmcblk0p1            8192      122879       57344    c  W95 FAT32 (LBA)
/dev/mmcblk0p2          122880    15523839     7700480   83  Linux
```

In my case, I know the SD card is 8GB in size, so the above difference in before and after leads me to conclude that `/dev/mmcblk0` is indeed my SD card.

The above listing is from a card that already has been used in my Pi, it has two existing partitions - which I'm about to wipe!

Now that we have a device, we can create the single partition we require:

```bash
fdisk /dev/mmcblk0
```

The above command takes you into the `fdisk` utility that allows you to manipulate (and destroy!) your SD card. It must be run as root, or using `sudo`.

If you have existing partitions on the device, delete them using the 'd' command. (Use the '**m**' command to get a list of available commands if you need to.)

```text
Command (m for help): d
Selected partition 1
```

You will be prompted for the partition number each time, so repeat until you have deleted them all. Listing the existing partitions afterwards should show the following:

```text
Command (m for help): p

Disk /dev/mmcblk0: 7948 MB, 7948206080 bytes
4 heads, 16 sectors/track, 242560 cylinders, total 15523840 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

        Device Boot      Start         End      Blocks   Id  System
```

Now we can create a single new partition, the '**n**' command does this:

```text
Command (m for help): n

Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p

Partition number (1-4, default 1): 1

First sector (2048-15523839, default 2048): 
Using default value 2048

Last sector, +sectors or +size{K,M,G} (2048-15523839, default 15523839): 
Using default value 15523839
```

The above creates a primary partition, numbered 1, with the default start and end sector values. This creates a single partition over the entire SD card, with a bit of space left - as required - at the start.

Use the '**p**' command to check all is well:

```text
Command (m for help): p

Disk /dev/mmcblk0: 7948 MB, 7948206080 bytes
4 heads, 16 sectors/track, 242560 cylinders, total 15523840 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

        Device Boot      Start         End      Blocks   Id  System
/dev/mmcblk0p1            2048    15523839     7760896   83  Linux
```

So far so good, except the partition type is `Linux` and not `FAT32`. We need to change it. The '**t**' command is our friend:

```text
Command (m for help): t
Selected partition 1
Hex code (type L to list codes): l

...
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor      
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi eb  BeOS fs        
...

Hex code (type L to list codes): b
Changed system type of partition 1 to b (W95 FAT32)
```

Again, use the '**p**' command to check all is well:

```text
Command (m for help): p

Disk /dev/mmcblk0: 7948 MB, 7948206080 bytes
4 heads, 16 sectors/track, 242560 cylinders, total 15523840 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

        Device Boot      Start         End      Blocks   Id  System
/dev/mmcblk0p1            2048    15523839     7760896    b  W95 FAT32
```

{{< alert theme="info" >}}
Some people may have problems getting their card to boot in the Raspberry Pi when wiped and re-partitioned like this. If you have that problem, simply make the new partition bootable using the **a** command.
 
I didn't need to do this on my card(s) but I have heard from people who do. Strange that some cards need bootable partitions while others don't.
 
When NOOBS' own installer takes over, it will partition the card to suit itself, and set bootable flags as desired. And it won't be the partition you created that NOOBS makes bootable, it will be a small one, located right at the very end of the card, which holds the NOOBS installer.
{{< /alert >}}

All that remains now is to write the new partition table to the SD card. The '**w**' command does this:

```text
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: If you have created or modified any DOS 6.x
partitions, please see the fdisk manual page for additional
information.
Syncing disks.
```

`Fdisk` will exit when we write the partition table.

So far, all we have done is to create a new, blank, partition and although we told `fdisk` that it would be `FAT32` we have yet to format it. We need to do this now, however, be careful to format the _partition_, `mmcblk0p1`, and not the _device_, `mmcblk0`.

```bash
mkfs.vfat /dev/mmcblk0p1

mkfs.vfat 3.0.12 (29 Oct 2011)
```

That's done when you get back to the prompt. Now we can get the software.

## Download the NOOBS Software

Go to [http://www.raspberrypi.org/downloads](http://www.raspberrypi.org/downloads "http://www.raspberrypi.org/downloads") and at the top of the page, you will notice the section entitled "New Out Of the Box Software (Recommended)" - that's where we need to be.

Click on either the torrent or direct download links. To save bandwidth on the main servers use the torrent link - if you have a torrent client installed, otherwise, click the direct link.

When prompted, save the file to a good location, I use `Downloads/RaspberryPi/distros` in my home directory. Your location might possibly be different!

It takes about 10 minutes to download, depending on your bandwidth and the busyness of the server. I was getting around 1800 Kb per second download speed, and the file itself is around 760 Mb in size, which is smaller than previous versions but it no longer has 6 different distros inside it!

While it's downloading, have a cuppa!

When it is finished downloading, it is best to verify that what you have is exactly what you should have. The download page shows a SHA1 checksum of, currently, "7bc2220f9c6a63cbcc6cafb039a3f0b24055cf23" so, open a shell session and verify the download:

```bash
$ # The following command is "sha(one)sum" and not "sha(ell)sum" :-)
$ sha1sum /home/norman/Downloads/RaspberryPi/distros/NOOBS_v1_3_11.zip 7bc2220f9c6a63cbcc6cafb039a3f0b24055cf23  /home/norman/Downloads/RaspberryPi/distros/NOOBS_v1_3_11.zip
```

You don't need to be root to do this, so `sudo` is not required. Compare the two checksums and if they are both the same, carry on, otherwise, the download is probably corrupt. Download it again.

## Copy to the SD Card

Now we need to mount the SD card, so eject it and put it back in again. This time, if you are prompted to open it in a file browser, do so. You can find out where it was mounted by running the `mount` command, as your normal user, and search from the partition name as above:

```bash
$ mount | grep -i mmcblk0p1
```

You should see something like:

```text
/dev/mmcblk0p1 on /media/F493-CE5C type ....
```

We need to be in a shell session in the above location, so :

```bash
$ cd /media/F493-CE5C
```

And now we are ready to extract the software to the SD card. This is a simple `unzip` command as follows:

```bash
$ unzip /home/norman/Downloads/RaspberryPi/distros/NOOBS_v1_3_11.zip 

Archive:  /home/norman/Downloads/RaspberryPi/distros/NOOBS_v1_3_11.zip
  inflating: bootcode.bin            
  inflating: BUILD-DATA              
   creating: defaults/
   creating: defaults/slides/
  inflating: defaults/slides/A.png   
  inflating: INSTRUCTIONS-README.txt  
   creating: os/
   creating: os/Raspbian/
  inflating: os/Raspbian/os.json     
  inflating: os/Raspbian/root.tar.xz  
  inflating: os/Raspbian/partitions.json  
  inflating: os/Raspbian/partition_setup.sh  
  inflating: os/Raspbian/release_notes.txt  
   creating: os/Raspbian/slides_vga/
  inflating: os/Raspbian/slides_vga/C.png  
  inflating: os/Raspbian/slides_vga/A.png  
  inflating: os/Raspbian/slides_vga/D.png  
  inflating: os/Raspbian/slides_vga/G.png  
  inflating: os/Raspbian/slides_vga/B.png  
  inflating: os/Raspbian/slides_vga/E.png  
  inflating: os/Raspbian/slides_vga/F.png  
 extracting: os/Raspbian/Raspbian_-_Boot_to_Scratch.png  
  inflating: os/Raspbian/flavours.json  
  inflating: os/Raspbian/boot.tar.xz  
 extracting: os/Raspbian/Raspbian.png  
   creating: os/Data_Partition/
  inflating: os/Data_Partition/os.json  
  inflating: os/Data_Partition/partitions.json  
 extracting: os/Data_Partition/data.tar.xz  
 extracting: os/Data_Partition/Data_Partition.png  
  inflating: recovery.cmdline        
  inflating: recovery.elf            
 extracting: RECOVERY_FILES_DO_NOT_EDIT  
  inflating: recovery.img            
  inflating: recovery.rfs            
  inflating: riscos-boot.bin  
```

That's it. The software is installed on the SD card, and is ready to be used. We need to get out of the SD card though, and unmount it before we can use it in the RaspberryPi.

```bash
$ cd
$ sudo umount /dev/mmcblk0p1 
```

## Installation of Your Chosen OS

Insert the SD card into your RaspberryPi, **with the power disconnected** and then connect or turn on the power.

A mini-system will appears and start reorganising the SD card layout. Partitions will be resized and new ones created and so on. You will end up with an SD card with a different set of partitions to the one single partition that we created above, this is ok. NOOBS is doing its thing and getting the SD card ready for installation.

Once the partitions have been resized and created as required, you are presented with a menu screen where you may select up to 7 different Operating Systems to install, plus an option to create a separate 512 Mb data partition.

At the bottom of the screen, there is an option to set the language to be used for the installation, and you may also select a keyboard layout here - if yours differs from the default.

Can't see the screen? Press one of the following number keys to assist, it may be best to press a digit key on the top row of the keyboard instead of the numeric keypad, which might not be activated by NOOBS by default.

- "1" to select output to the HDMI socket, and potentially, from there, to a VGA adaptor. This is the default output for NOOBS.
- "2" to select output to "Safe HDMI". Choose this if the default HDMI isn't showing anything and you are definitely using an HDMI connected monitor or TV.
- "3" to select composite PAL output. This is the "normal" TV arial socket on the side of your Pi, if your Pi is as old as mine! Later Pis have different sockets for composite output. PAL is used in the UK, Japan and Australia, for example.
- "4" to select composite NTSC for the USA TV system. I'm told it stands for Never Twice the Same Colour!

The Available Operating Systems are:

- Raspbian
- Boot to Scratch
- Arch
- OpenELEC
- RaspBMC
- PiDora
- RiscOS

There is an additional option to create a 512 Mb data partition. This could be useful, and If you have sufficient space on the SD card, I would advise selecting this option, especially if you may be installing more than one Operating System.

Note: The operating systems shown with an SD Card icon are available directly from the NOOBS card. Those with an icon resembling a network cable and network socket are required to be downloaded from the internet, so you will need a working internet connection to install Arch, for example.

Select all the desired Operating Systems you wish to install by clicking the small box at the start of each entry in the menu. If a tick or cross appears, it has been selected. Just click it again if you made a mistake, or have changed your mind.

When done selecting Operating Systems, click on the "Install" button. Take note of the warning that appears next - if you are planning to try another distro, then any existing data on this SD card will be obliterated when the new OS is installed. This isn't a problem on the very first use of the SD card, but might be if you have been using one particular distro for a while but wish to try another.

If you need to preserve your data, make a backup of at least your Home directory, and anything in the separate data partition, just to be safe.

Click the "yes" button to install the OS. Then go and have another coffee because it takes a wee while to write the OS to the SD card. My own Pi gives me about 1.8 MB per second and there is quite a few Gb to write - depending on how many Operating Systems are being installed.

The installation will display a running total of the amount of data written so far, and how much is to be installed in total. Beware that these numbers do not include the Operating Systems that must be downloaded, only those that are on the SD card itself.

When all the installations have completed, a pop-up dialogue will let you know that it's time to reboot. Click the OK button on the dialog that has the title "OS(es) Installed" and the Pi will reboot. If you installed more than one Operating System, a menu will let you select the one you wish to boot into. Double-click the chosen Operating System and the Pi will boot that one.

You might have some initial configuration to do on first use, that depends on the OS in question.

## Selecting a New Operating System on Reboot

When the Pi reboots, if you installed more than one Operating System, you will be offered a menu of the installed Operating Systems. As above, double-click the desired one. If you do nothing for 10 seconds, the Operating System you used last will be booted.

If you created a separate Data Partition, where is it? A good question. My own install of Raspbian, Arch, OpenElec, Riscos and a Data Partition, resulted in my 16 Gb SD card being formatted with 13 different partitions, set up as follows:

- RiscOS on partitions 5 and 6.
- Data Partition on partition 7.
- Raspbian on partitions 8 and 9.
- Arch Linux on partitions 10 and 11.
- OpenElec on partitions 12 and 13.

So the data partition is partition 7, and we know that it is formatted as an ext4 file system, so where is it mounted when I boot into Rasbian, for example?

The `mount` command is your friend and it can be run from your normal pi user, not just root. Running the command, results in the following:

```bash
$ mount | grep -i p7

/dev/mmcblk0p7 on /media/data type ext4 ...
```

So, it would appear that the data partition is mounted on /media/data. In fact, _all_ the other Operating Systems' partitions are automatically mounted by Raspbian, in `/media/something_or_other`.

When logging in to Arch Linux, the data partition is not automatically mounted, and neither are any of the other Operating Systems' partitions. In Arch Linux, _you_ tell it what to do - it makes no assumptions.

## Reinstallation and Recovery

If you need to reinstall, or recover from a huge foul up, you should first attempt to backup any data that you have been creating. The recovery process will trash it without trace.

Of course, if the system is so badly trashed, you might not be able to take a backup, but at least try!

Boot the RaspberryPi from the SD card again, and when prompted on screen (for about 3 seconds) press and hold the SHIFT key. Any one will work.

The installation menu will appear again, simply follow the instructions in the _Installation_ section above to recover your OS. Alternatively, choose a new OS to try out - there are 7 to choose from, as listed above.

Have fun.
