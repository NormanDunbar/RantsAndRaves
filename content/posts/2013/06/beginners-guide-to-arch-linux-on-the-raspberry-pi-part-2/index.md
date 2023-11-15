---
title: "Beginner's Guide to Arch Linux on the Raspberry Pi - Part 2"
date: "2013-06-12"
categories: 
  - "linux"
  - "raspberry-pi"
---

Continuing with the setting up and such like, using Arch Linux on the Raspberry Pi. This is the second blog post on the subject.

In the [previous posting](http://qdosmsq.dunbar-it.co.uk/blog/2013/06/beginners-guide-to-arch-linux-on-the-raspberry-pi/ "Beginner’s Guide to Arch Linux on the Raspberry Pi"), we managed to set up our locales, languages, keyboards, and so on. We have internet connection vie a wired Ethernet connection using either a dynamic IP address or a Static one. Time to move on.

As before, if you see a command with a '**>**' prompt it means that you should be in the root user, or have prefixed the command with `sudo`. If the prompt is '**$**' then that runs as your normal 'pi' user - which we created last time.

## WiFi Access to the Internet

Setting up a WiFi access is just as simple as setting up the Wired access, actually, it's easier as there is a utility that helps out.

Make sure your WiFi dongle is plugged in and then:

```bash
> wifi-menu -o
```

The last parameter is a lower case letter o, not a zero.

After a short delay, you will see a message telling you that the system is "scanning for networks". Eventually, a list of available networks will be displayed - if yours is not showing, make sure that your wireless router is broadcasting the SSID. Follow the router's instructions if you need to make changes.

Use the arrow keys to move up and down the list of networks until you have highlighted the desired one. Press the return key to select it.

If the network is password protected, you will be prompted for the password. Type it in and press return to accept it. If the network is not protected - you are asking for trouble!

After a pause, you will get the shell prompt back again. You should now be connected to the internet which you can check, as before, by pinging 8.8.8.8 to see if the connection is talking to the internet and then ping google.com to ensure that you have name resolution too. Both are required.

The wifi-menu utility has created a new profile for your WiFi link. It is called wlan0-SSID where the SSID part of the name is the name of the network you connected to.

It has been configured for a dynamic IP address, and you might not want this.

To set up a static IP address, proceed as follows:

```bash
> cd /etc/netctl
> vi wlan-SSID      # Where SSID is your network name
```

You will see something like the following:

```text
Description='Automatically generated profile by wifi-menu'
Interface=wlan0
Connection=wireless
Security=wpa
ESSID=your_network_name
IP=dhcp
Key=abc1234dhdjfkfkfkgjflsdjjbskbk.bkjbvkbkjbvkjvkbvkohu
```

You need to change it to something like the following:

```text
Description='Automatically generated profile by wifi-menu'
Interface=wlan0
Connection=wireless
Security=wpa
ESSID=your_network_name
#
# Changes below here.
#
IP=**static**
Key=abc1234dhdjfkfkfkgjflsdjjbskbk.bkjbvkbkjbvkjvkbvkohu
Address=('192.168.1.36/24')
gateway=('192.168.1.1')
DNS=('8.8.8.8' '194.168.4.100' '194.168.8.100')
```

You will note that the IP address I'm using here is exactly the same as the one I set up in the last article. This is fine as I only ever user the WiFi or the wired connection, never both.

Plus, I don't have the network connection enabled to auto start at boot time.

Once the file is saved, you can start the WiFi network by:

```bash
> netctl start wlan0-your_network_name
```

If you always want the WiFi network to be started when the Pi boots, then enable it, as follows:

```bash
> netctl enable wlan0-your_network_name
```
{{< alert theme="info" >}}
If you sometimes use a WiFi with a static IP address - say at home, but use a dynamic one when out and about - say at a coding class or Raspberry Jam etc, then you can set up two separate profiles.
 
Use the `wifi-menu` command to set up the dynamic profile, and rename it to `wlan0-your_network_name.dynamic` then copy it to `wlan0-your_network_name.static` and add in the bits you need, as above, for a static IP address.
 
The `netctl list` command will list both and you can start and stop either one at will.
 
Obviously, this only applies if the two profiles relate to the same network. If you have a home network called `home` and a work one called `work` they will already be different files!
{{< /alert >}}

## Update the System

Now that we have a working system, it's time to update the system. The package system on Arch Linux is called `pacman` and is quite simple to use:

```bash
> pacman -Syu --ignore filesystem --ignore ca-certificates
```

The above command first of all updates the current database with any changes since the release of the version you are running, then checks to see what needs updating. After a few seconds, you are presented with a list of packages that are installed, but need updating. Type **Y** to update the lot. Time for a coffee I suspect!

{{< alert theme="info" >}}
Note: When I upgraded my system, I saw two errors. One in package `filesystem` and the other in `ca-certificates`. Everything else installed correctly. The above command actually ignores those two packages completely.
 
Once everything else installed, I simply ran the following command and the two ignored packages were upgraded without error:
 
```bash
> pacman -Syu
```
{{< /alert >}}


If you see a message, during the upgrading of the system, which mentions that "/`etc/pacman.d/mirrorlist` has been saved as `/etc/pacman.d/mirrolist.pacnew`" then you might need to do a manual correction.


```bash
> cd /etc/pacman.d
> grep -v ^# mirrorlist

Server = http://mirror.archlinuxarm.org/armv6h/$repo

> grep -v ^# mirrorlist.new

Server = http://mirror.archlinuxarm.org/armv6h/$repo
```

As you can see, both files contain a single uncommented line, which is the pacman repository to use on a Raspberry Pi. As they are the same, we can delete the old file and rename the new one with no problems:


```bash
> rm mirrorlist
> mv mirrorlist.pacnew mirrorlist
```

If there were other uncommented lines in either of the files, you would need to make the `mirrorlist.pacnew` file match the existing `mirrorlist` file prior to deleting and renaming.

You now have an up to date `mirrorlist` file.

## Graphical User Interface

Arch doesn't come with much already installed, just the base system. This keeps bloat to a minimum, and allows _you_ to decide what goes into your own setup. Arch also doesn't come with a GUI system at all, so now, we can install one if we wish.

The Linux GUI system comes in three parts:

- X-Org
- Window Manager
- Desktop Environment

The file `/usr/share/X11/xkb/rules/xorg.lst` contains details of various settings that you can use for your keyboard _model_, _variant_ and language (aka _layout_) and such like.

The default for my keyboard model and a possible variant of that model (see the above file for details), is correct, but my layout is wrong. It is set to US while I want GB. (Yes, in the shell it's UK and in X it's GB, go figure!)

To fix the problem of a "broken" layout, we need to create a file called `/etc/X11/xorg.conf.d/10-keyboard.conf` and add the following to it:

```text
Section "InputClass"
    Identifier             "Keyboard Defaults"
    MatchIsKeyboard	   "yes"
    Option	           "XkbLayout" "gb"
EndSection
```

Obviously, if you want a French keyboard layout, you would set it accordingly. I need "gb" for mine.

If your keyboard model and/or variant isn't correct, you may also add one or both of the following lines after the existing "option":

```text
    Option                 "XkbModel"   "xxxxxx"
    Option                 "XkbVariant" "yyyyyy"
```

Where 'xxxxxx' is one of the models, and 'yyyyyy' is one of the variants listed in /`usr/share/X11/xkb/rules/xorg.lst`. If you now run the `startx` command and start typing shift-3 etc, you should see that your keyboard is correctly configured. If not, make sure you spelt the various options correctly - and not as I originally did - for example, there is no 'kbd' in 'Xkblayout' etc, it's just 'kb'!

Another way to set the keyboard layout and such like is to run the following command in the X session when you have started it, or add it to the file `~/.xinitrc` in your particular user:

```bash
> setxkbmap -layout "gb" -model "xxxxxx" -variant "yyyyyy"
```

Where the layout, model and variant are as desired. You may omit one or other of these if the default settings are correct.

Next, we need a Window Manager, and for this experiment in Arch, I'm installing `Openbox` - it's lightweight and "fun".


```bash
> pacman -S openbox obconf obmenu lxappearance
```

When the above has completed, running `startx` will have apparently made no difference. We need to tell the X system to run Openbox on startup, so, edit the file `.xinitrc` in your home directory (`/home/root` as we are working as the root user at present) and make it thus:

```text
openbox-session
```

Also, in order to have at least something on the openbox context (ie, right-click) menu, do the following as well:

```bash
> mkdir -p ~/.config/openbox/
> cp /etc/xdg/openbox/menu.xml ~/.config/openbox
> cp /etc/xdg/openbox/rc.xml ~/.config/openbox
```

When you create a new user on the system, you will need to do a similar exercise for those user accounts, like pi, that you wish to run a GUI on login.

Now, run the `startx` command and be amazed! You get, as I did, a totally black screen with an arrow pointer on it. Congratulations, you are now running an openbox window manager.

Right-click the black area to see the default context menu, select `terminals` and select `xterm` from the list. You should now see a new window appearing running a terminal session. You can click on the caption bar and move it around and such like.

When you exit from xterm, you are back at the blank desktop. The context menu allows you to configure open box, if you wish to do so, right-click, select `System` and `Openbox Configuration Manager` to play around.

Select `Log Out` and `Exit`, when prompted, to exit from openbox.
