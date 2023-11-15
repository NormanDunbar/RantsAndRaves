---
title: "Backup & Check a Raspberry Pi SD Card"
date: "2012-10-22"
categories: 
  - "linux"
  - "raspberry-pi"
---

Ever wanted to backup your Raspberry Pi's SD card, but didn't know who to ask? Me too. Read on ....

The first thing to remember is that you should really always have a backup of your SD card. In theory every time you make a change, but in practice, it will be less frequently than that!

I have had two cards corrupt themselves when I managed to lock my Pi completely, and the only recourse was to pull the power. Unfortunately, in both cases, I was left with a card that the Pi could read the first partition from, but then couldn't mount the second, so no operating system.

Luckily I had a backup, but it was of a 4Gb SD card and I'm running on 8 Gb, so I hjad to restore and then extend the second partition to use the remaining 4 Gb of unallocated space. Normally I'd restore the 8 Gb image to an 8 Gb card.

These instructions are for Linux only. If you use one of the Windows utilities for writing to your SD card, there should be an option to read from it and write to a file, if so, this is what you need to use. Beware, there may need to be two reads if you have two partitions, with some of these utilities.

## Make a Backup

Plug your card into your computer, no, not your raspberry Pi ;-), and wait a second or two for it to settle down.

If you have a card slot, you most likely have the card on `/dev/mmcblk0` and if you have it in a USB adaptor of some kind, it's most likely going to be `/dev/sdx` where 'x' is the digit representing the highest device of this kind.

To find out for certain, do the following, either using `sudo`, or as root:

```bash
fdisk -l
...
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

The above shows the tail end of the output on my system. I can recognise the card by the size, 8 Gb (or 7948 as listed above) which is the only drive I have of this size, and its two partitions bearing in mind the fact that one of them is small and W95 FAT (LBA) and the other plain old Linux.

In the above, the suffixes 'p1' and 'p2' indicate the partitions on the card. The device itself is `/dev/mmcblk0` and that's what we need, the device name, not the partition names.

To take a backup, switch to your backup directory, where you intend to keep the copy of your SD card, and run the following command as the root user (or use `sudo`):

```bash
dd if=/dev/mmcblk0 of=Rpi_8gb_backup.img bs=2M 
3790+0 records in
3790+0 records out
7948206080 bytes (7.9 GB) copied, 1369.42 s, 5.8 MB/s
```

That's it. The whole 8Gb SD card has been (slowly) copied to my computer. How do I know it worked?

## Check the Backup File

The output above shows that there were no errors in copying the SD card to disk, but it's always best to be sure that it will work when the time comes to carry out a restore.

The first step, as root - as ever - is to determine where the partitions begin. Because we have an image copy of the SD card, we can look at the file itself and see the partition table within.

The following commands will be carried out as root, or using the `sudo` command.

We first list the SD image file to see where the partitions begin and the sizes of the sectors. `Fdisk` reports sizes in sectors, but handily displays the sector size at the top.

```bash
fdisk Rpi_8gb_backup.img 
Welcome to fdisk (util-linux 2.21.2).
...
Command (m for help): p

Disk /home/norman/Backups/Rpi_8gb_backup.img: 7948 MB, 7948206080 bytes
255 heads, 63 sectors/track, 966 cylinders, total 15523840 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000dbfc6

                                  Device Boot      Start         End      Blocks   Id  System
/home/norman/Backups/Rpi_8gb_backup.img1            8192      122879       57344    c  W95 FAT32 (LBA)
/home/norman/Backups/Rpi_8gb_backup.img2          122880    15523839     7700480   83  Linux
```

We can see from the line starting with 'Units' or 'Sector size' that a sector is 512 bytes. We can also see that our two partitions start at sectors 8,192 and 122,880.

Multiplying these start sectors by 512 gives the start position in bytes. This works out at 4,194,304 and 62,914,560 bytes respectively.

On your Raspberry Pi, the first partition is mounted at `/boot` while the second is the root (/) mount point.

To check these, without needing access to the Pi, we simply proceed as follows on a Linux computer:

```bash
mkdir /mnt/root /mnt/boot
chmod a=rwx /mnt/root /mnt/boot

mount -t vfat -o loop,offset=4194304 /home/norman/Backups/Rpi_8gb_backup.img /mnt/boot

mount -t ext4 -o loop,offset=62914560 /home/norman/Backups/Rpi_8gb_backup.img /mnt/root
```

The above creates a couple of mount points (directories) and then mounts the first partition within the image file as a `vfat` file system on `/mnt/boot` and then mounts the second partition as an `ext4` file system on `/mnt/root`.

The first two commands to create the mount points and set the permissions on them are only required once, the first time you carry out this exercise.

You can now see the files by opening a file manager and looking at the `/mnt/boot` and `/mnt/root` directories - you should see your various files as if you were looking on your Raspberry Pi.

## Restore a Single File

This method of mounting a disc image as a device is useful for times when you manage to delete a file, accidentally of course, on your Pi, but you know that you have a backup. If the Pi is on your network, you can simply mount the backup image as above, locate the file and check that it is the one you want, then use `scp` or `sftp` to copy the file to your Pi, as demonstrated below:

```bash
mount -t ext4 -o loop,offset=62914560 /home/norman/Backups/Rpi_8gb_backup.img /mnt/root

scp /mnt/root/etc/network/interfaces pi@raspberrypi:
pi@raspberrypi's password: 
interfaces          100%  183     0.2KB/s   00:00
```

The file is now located in the home directory of the pi user. All you need to do to move it to the correct place is:

```bash
sudo mv interfaces /etc/network/
sudo chmod u=rw,go=r /etc/network/interfaces
sudo chown root:root /etc/network/interfaces
```

Obviously, if you are like me, you have already given your root user a password, so you could miss out the above and simply `scp` the file straight into the correct location. You would only need to set the permissions and not the ownership of the file afterward.

```bash
scp /mnt/root/etc/network/interfaces root@raspberrypi:
root@raspberrypi's password: 
interfaces          100%  183     0.2KB/s   00:00

sudo chmod u=rw,go=r /etc/network/interfaces
```

So there you have it, how to backup your SD card, how to mount it on the backup computer to check that it is ok, and as a bonus, how to extract a file (or files) for an individual recover. What else could we do?

## Restore a Full Backup

This is as simple as initialising the SD card for the first time. You can restore a backup file to your SD card provided that the SD card is bigger or the same size as the image file.

As ever, the following commands are executed as root or by using the `sudo` command.

```bash
dd if=Rpi_8gb_backup.img of=/dev/mmcblk0 bs=2M
```

It takes a while, but it will restore in the end. All you have to do is wait for the copy to finish and then boot the Pi with the restored SD card in the slot.

## Copy Your Current System to a Larger Card

We can also use the smaller backup files to initialize a larger SD card, if we were perhaps upgrading. This is what I had to do when I restored a 4 Gb backup to my 8 Gb card. Once that was done, I booted the Pi with the restored SD card in place, and when I logged in, executing `sudo raspi-config` and selecting the option to `Expand root partition to fill SD card` extended the current almost 4 Gb partition to the full almost 8 Gb size required (on the next reboot that is.)

You can see that it worked by executing the `df` command, which does not need to be executed as root!

```bash
df -h
Filesystem      Size  Used Avail Use% Mounted on
rootfs          7.3G  1.6G  5.4G  23% /
/dev/root       7.3G  1.6G  5.4G  23% /
devtmpfs        109M     0  109M   0% /dev
tmpfs            22M  208K   22M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            44M     0   44M   0% /run/shm
/dev/mmcblk0p1   56M   25M   32M  44% /boot
```

The first and last entries in the above show that the root file system, mounted on `/` is 7.3 Gb while the `/boot` file system is 56 Mb in size. So, my 4 Gb restore is now happily running on an 8 Gb card and I have all the space available to use.
