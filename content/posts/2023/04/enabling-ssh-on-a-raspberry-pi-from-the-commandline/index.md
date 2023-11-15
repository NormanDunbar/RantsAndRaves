---
title: "Enabling SSH on a Raspberry Pi from the Commandline"
date: 2023-04-14T15:06:07+01:00
lastmod: 2023-04-14T15:06:07+01:00
description: "How to Enable SSH on a Raspberry Pi from the Commandline."
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocFolding: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
categories:
    - "Raspberry Pi"
    - "Volumio"
    - "SSH"
    - "SSHD"
---

The [Raspberry Pi](https://www.raspberrypi.com/ "https://www.raspberrypi.com/") is a great little device. I use one to play my music as my CD player died a death a while back -- the CDs kept slipping -- and spares are very hard to come by as my "stack" is over 30 years old, but still sounds great. I have ripped my CDs, or most of them, to FLAC format, and have some of them stored on a couple of larger sized USB thumb drives attached to a Raspberry Pi. The Pi has a [HiFiBerry DAC Plus](https://www.hifiberry.com/dacs "https://hifiberry.com/dacs") attached -- which I was lucky enough to win in a [MagPi Magazine](https://magpi.raspberrypi.com/ "https://magpi.raspberrypi.com/") giveaway -- so the sound is excellent played through my Sony "stack".

I recently had to update [Volumio](https://volumio.com "https://volumio.com") to version 3, and while I'm still happy with it, I do now have a couple of foibles, and bugs which seemingly were fixed still appear to be not fixed. I couldn't change the theme, for example, until I logged onto the Raspberry Pi and executed the command `touch /data/manifestUI`.

As it turned out, changing the theme was a nightmare as there is (now) a transparency applied so that background shows through the text. This is garbage! Especially on the "classic" theme, the one I actually like, but there no longer seems to be an option to select a better background, and to lose the transparency.

Anyway, I also wanted to stream my music via [MStream](https://mstream.io/ "https://mstream.io/"), so I needed to install and enable SSH, so that I could login and install software remotely. This  turned out to be *interesting* as the Pi uses a different name for the SSH daemon, it's `ssh` rather than `sshd` which is on every other Linux installation that I know of! Also, because of how the Volumio installation takes over the Pi, I was unable to install *nodejs* and *npm* which are requirements of MStream. Bummer! I still needed remote access to the Volumio server though.

To install the SSH server, if not already installed:

```bash
sudo apt update
sudo apt upgrade
sudo apt install openssh-server
```

If it needs to be installed, it will be, otherwise you will be told that it's already there. Handy.


Once installed, you need to start it:

```bash
sudo systemctl start ssh    ### Beware, 'SSH' not 'SSHD'!
```

Test it by logging into your Pi from a different system. If its ok, then enable it to automatically start at boot time:

```bash
sudo systemctl enable ssh
```

Job done. After a reboot -- to test the automatic starting of the server -- you should be able to login.


If necessary, generate a new key file with:


```bash
ssh-keygen -t ecdsa -b 521
```

This will generate a 521 bit ECDSA key pair. These days, the DSA key type is best ignored and RSA is *probably* going to be broken at some point soon. 

The key files, for there are two, generated will be in your `$HOME/.ssh` directory:

```bash
ls ~/.ssh/id*

...
/home/norman/.ssh/id_ecdsa
/home/norman/.ssh/id_ecdsa.pub
...
```

The new key file can be copied to the server now:

```bash
ssh-copy-id -i ~/.ssh/id_ecdsa volumio@wvolumio
```

Obviously, substituting your key file name and logon details, you will be prompted for a password at this stage. Once copied over, test that it worked:

```bash
ssh volumio@wvolumio
```

And this time, if all was well, you will get logged straight in without needing a password.