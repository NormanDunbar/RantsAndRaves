---
title: "Installing Oracle Java 7 on Linux Mint 13"
date: "2013-01-11"
categories: 
  - "linux"
---

I use Java only when I have to, and only ever the JRE (Java Runtime Environment) - there is no way I'll use Java for development work. I'd rather eat my own ear wax to be honest!

Linux Mint 13 comes with OpenJDK installed, and a system I use which runs the `fop` FO Processor, barfs with a Java.Lang.NullPointer Exception if OpenJDK is used.

The following is brief instructions on how to install Oracle's java on Linux Mint.

## Download Java JRE

Go [to here](https://www.java.com "Javacom - java downloads") and click on the "Free Java Download" button.

Click on Linux or Linux-x64 depending on whether you are running a 32 or a 64 bit Linux system. When prompted, select a suitable place to save the file, and click OK.

Remeber, you don't want the various Linux*.rpm packages - they are for Red Hat/Fedora/Scientific Linux/Centos/OpenSuse/Oracle Linux distributions.

When the download is complete, you should have a saved copy of something like `jre-7u10-linux-x64.tar.gz`

## Install Java

In a shell session (don't be afraid!), change to the directory where you saved the downloaded file. In my case, that was Downloads/Java in my home directory. Then uncompress the file.

```bash
$ cd ~/Downloads/Java
$ tar -xvzf jre-7u10-linux-x64.tar.gz
```

Lots of filenames will whizz past on the screen. Wait for the process to finish.

You now need to be root, so, use `sudo sh` or `su -` or whatever to get you into a root session. I use `su` as my root user has been given a secure password.

In a root session, you need to move all the files you just extracted to the /usr/lib/jvm

```bash
$ su -
password:

$ cd ~norman/Downloads/Java
$ ls
jre1.7.0_10  jre-7u10-linux-x64.tar.gz

$ mv jre1.7.0_10 /usr/lib/jvm/
```

That's all done now, Java is in the correct place. All that remains is to configure the system to use it in preference to anything else that is present.

Still working as root, run the `update-alternatives` command, as follows, to tell the system about the new Java version, and to set it as default.

```bash
$ update-alternatives --install /usr/bin/java java /usr/lib/jvm/jre1.7.0_10/bin/java 1065

update-alternatives: using /usr/lib/jvm/jre1.7.0_10/bin/java to provide /usr/bin/java (java) in auto mode.
```

And also the following:

```bash
$ update-alternatives --install /usr/bin/javaws javaws /usr/lib/jvm/jre1.7.0_10/bin/javaws 1065

update-alternatives: using /usr/lib/jvm/jre1.7.0_10/bin/javaws to provide /usr/bin/javaws (javaws) in auto mode.
```

That's it, Oracle's Java is now installed and will be used as the default.

If, for some strange reason, I had installed a full development kit, rather than just the runtime, I would have required the following two commands to be run as well - to configure the default java compiler and jar tool.

```bash
$ update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jre1.7.0_10/bin/javac 1065
$ update-alternatives --install /usr/bin/jar jar /usr/lib/jvm/jre1.7.0_10/bin/jar 1065
```

## Changing the Default Java Version

The above makes my development tool happy as `fop` no longer barfs with an exception and my FO source code is happily converted into pdf documents.

What happens when some other program doesn't like to play with Oracle's Java and needs the original OpenJDK version instead?

Simple, change the default again:

```bash
update-alternatives --config java
There are 3 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/jre1.7.0_10/bin/java                1065      auto mode
  1            /usr/lib/jvm/java-6-openjdk-amd64/jre/bin/java   1061      manual mode
  2            /usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java   1051      manual mode
  3            /usr/lib/jvm/jre1.7.0_10/bin/java                1065      manual mode
```

Press enter to keep the current choice[*], or type selection number: 

The `update-alternatives` displays all known installed versions of Java and lets you choose the one you want. The first entry is the current one, and you simply type in the number of the one you want instead.

Have fun. (If there's such a thing as fun where Java is involved!)
