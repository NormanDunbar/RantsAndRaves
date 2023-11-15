---
title: "Backing Up My Books"
date: "2022-08-12"
categories: 
  - "android"
  - "linux"
  - "personal"
---

I have a lot of eBooks on my tablet and phone. I also have a backup on my Linux laptop. In the past I have often just plugged the tablet into the laptop, opened the appropriate MTP folder, selected everything and copied them all to the backup, overwriting the previous backup. There has to be a better way, surely? Step up `rsync`.

The problem is the MTP mount, however, I found a way to make it work. Those mounts, mainly from Android devices, usually open the the appropriate file explorer application in the laptop. But, they are mounted -- somewhere -- and with a bit of digging, we can find the mount point, and from that, run `rsync` to do the backup of the files that are new or updated only.

## Finding the Mount Point

The first problem, is where on earth has the Android device been mounted?

```
$ gio mount -li | grep -i activation_root |\
  cut --delimiter== --fields=2 |\
  cut --delimiter="/" --fields=3


SAMSUNG_SAMSUNG_Android_R52M70B1KWE/
```

So, that's the mount point sorted. Well, part of it. I need to know where it's mounted. This turned out to be:

`/run/user/1000/gvfs/mtp:host=SAMSUNG_SAMSUNG_Android_R52M70B1KWE/`

where 1000 is my user id. That colon will need escaping too, I suspect! Now, how to work that out in code in case it ever changes?

## Finding my User Id

This is easy enough, the file `/etc/passwd` holds details of my user id in the third field, fields are separated by colons, so, `cut` to the rescue!

```
$ grep ^$USER /etc/passwd |\
  cut --delimiter=: --fields=3

1000 
```

That was simple. So, this should work then? I'm using the short form of the `cut` parameters here by the way. Less typing! I'm also using the `$(...)` form of extracting a result from a command, Wordpress has stopped me using backticks for some reason!

```
$ ## Where the backups live.
$ cd ~/Backups/Tablet/

$ ## Get my user id,
$ USER_ID=$(`grep ^$USER /etc/passwd | cut -d: -f3`)

$ ## Get the MTP mounted device.
$ DEVICE=$(gio mount -li | grep -i activation_root |  cut -d= -f2 | cut -d/ -f3)

$ ## Get the actual mount point for the MTP device.
$ MOUNT=/run/user/${USER_ID}/gvfs/mtp\\:host\\=${DEVICE}/Tablet/My_Books

$ ## Run the backup from MTP to Backups/Tablet/My_Books/
$ ## Trailing slash on "$MOUNT" is important here.
$ rsync $MOUNT/ --archive --recursive --verbose My_Books/
```

And, indeed it does. I did have a problem though. `My_Books` used to be named `My Books`. Neither the `ls` command, nor `rsync` could see it on an MTP mount point, for some unknown reason. The file manager could, and allowed manual copying. However, a quick rename and all was well.

So, now I've built the above into a shell script and saved it in `$HOME/bin`/`backup_mybooks.sh` for future use.
