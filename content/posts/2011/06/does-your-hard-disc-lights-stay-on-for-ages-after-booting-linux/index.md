---
title: "Does Your Hard Disc Lights Stay On For Ages After Booting Linux?"
date: "2011-06-20"
categories: 
  - "linux"
---

Mine did, not any more though. Read on.

I was just a tad annoyed at the sudden increase in s-l-o-o-o-o-w responses after starting OpenSuse 11.4 up. I noticed that the hard drive indicator LED was continually lit for about 5-10 minutes after bootup. Performance was abysmal during this time.

Top showed almost no CPU being used, so I looks ad `vmstat` instead. This was the result, on an idle (and slow) system:

```bash
vmstat -d 

disk  ... -----IO------ 
      ...    cur   sec 
sda8  ...      0   898
```

{{< alert theme="info" >}}
In the above, I've removed the individual reads and writes columns, to sort out the wheat from the chaff. I'm interested in I/Os overall. I've also removed the partitions that were showing 0 or 1 as the total I/O count.
{{< /alert >}}

The figures showed me that partition sda8 was getting hit to the tune of almost 900 I/O operations per second. Who or what is partition sda8?

```bash
df -h | grep -i sda8 

/dev/sda8 251G 131G 108G 55% /data`
```

Now I know, so what's open on this mount point?

```bash
lsof /data 

COMMAND PID USER NODE NAME 
preload 385 root /data/VirtualBox/ScientificLinuxEnterprise6.vdi 
preload 385 root /data/VirtualBox/LinuxMint64bit.vdi
```

And having noted the above, why on earth was anything loading my virtual drives when I'm not even running VirtualBox at the moment?

```bash
ps -ef | grep 385 

root 385 1 1 11:15 ? 00:00:11 /sbin/preload 
/var/cache/preload/prepared
```

A quick check of the internet showed that `preload` is a process to save boot time (how ironic) and that it can easily be disabled. So I disabled it. No more long boots now! To disable preload, run `yast` and select `System Services (Runlevel)`.

- Tick the `Expert Mode` radio button.
- Scroll down to find `boot.startpreload`
- At the bottom of the screen, uncheck the `B` box.
- Click OK.

Cheers,  
Norm.
