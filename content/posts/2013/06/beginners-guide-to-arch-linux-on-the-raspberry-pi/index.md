---
title: "Beginner's Guide to Arch Linux on the Raspberry Pi"
date: "2013-06-04"
categories: 
  - "linux"
  - "raspberry-pi"
---

The ARCH Linus distro for the Raspberry Pi is not the normal one used by the masses, but the benefits of ARCH are good in that it is a rolling release distro. That means, you never have to reinstall it to be on the latest version.

The information that follows assumes that you have installed ARCH from the NOOBS installer. The latest version of ARCH has changed the networking system in use.

In addition, where you see a bash prompt that looks like **>** then the commands that follow whould be executed as the root user. Normal user commands have a **$** prompt.


## Sorting Out Your Keybord and Locale

By default, your keyboard is defined to be a US one. That's fine if you live in the USA, but for us Brits, and anyone else, that's no good.

You can check if it's configured correctly, by typing SHIFT+3 to get '£', you will most likely get '#' if the US keyboard is in use. For a quick fix for the current session, this will work:

```bash
> loadkeys uk
```

If you try typing a '£' again, it should be a '£' this time. if you are in another country, try entering your country code. You might find it useful to have a peek at the contents of the directory `/usr/share/kbd/keymaps/i386/` where you will find further directories related to your keyboard layout - qwerty, azerty, dvorak etc. The file names will give you a clue as to the code to use.

To make the UK keyboard, or whatever you have chosen to use, stick over a reboot, you need to edit the `/etc/vconsole.conf` file using `vi`, or `nano` or whatever editor you prefer. You should change only the line that begins with 'KEYMAP' as follows:

```text
KEYMAP=uk
```

Don't change any of the other lines. Save the file and exit from the editor.

Moving on from the keyboard, we now need to fix the system language. By default it is configured to be US English plus UK English. Well, speaking as a Brit, I'm not having any of that! ;-)

First let's check:

```bash
> grep -v ^# /etc/locale.gen
```

The command above lists all the uncommented lines from the `/etc/local.gen` file. That shows the locales that have been set up by default. On my Raspberry Pi, I see the following:

```text
en_US.UTF-8 UTF-8
en_GB.UTF-8 UTF-8
```

Edit the `/etc/locale.gen` file and comment out the en_US line by prefixing it with a '#'. If you are in France, for example, and wish to change the locale to your own, then comment out both lines showing above, and uncomment the one that reads "fr_FR.UTF-8 UTF-8'.

Save the file and exit from the editor. Now generate the locale files as follows:

```bash
> locale-gen
```

You should see something like the following:

```text
Generating locales...
  en_GB.UTF-8... done
Generation complete.
```

Now, set the current session's locale to the one just generated:

```bash
> export LANG=en_GB.UTF-8 
```

And finally, to make sure that `LANG` is correctly set over a reboot, edit the `/etc/locale.conf` file to the following:

```text
LANG=en_GB.UTF-8
LC_COLLATE=C
```

## Change the Root Password

Now that the keyboard and languages has been sorted out, we really must change the default root password. The current password is hugely insecure.

Type the command `passwd` and follow the prompts, as shown below:

Enter new UNIX password: 
Retype new UNIX password:
passwd: password updated successfully

That's that taken care of. Make sure you choose a secure password and I advise staying away from 'raspberry' or 'root', just in case.

## Configuring the Network (Wired Ethernet)

The latest release of Arch changed the way that networking is used from `netcfg` to `netctl`. In order to get a working network, you need to set up a parameter file.

I found on my own set up, that networking wasn't even enabled after I booted up, which isn't very useful.

Test if networking is running:

```bash
> ping -c 3 8.8.8.8
```

If you get a response telling you that you received '64 bytes from 8.8.8.8 etc' then you do already have a network that is running.

Now, check if you have DNS working:

```bash
> ping -c 3 google.com
```

This time, you should get a response telling you that you received '64 bytes from (173.194.41.135) etc. If so, your networking is already set up and you need do no more.

If neither of the above worked, then, fix the problem:

```bash
> cd /etc/netctl
> cp examples/ethernet-dhcp ./eth0
```

You should now be able to start the network with:

```bash
> netctl start eth0
```

On my system, that always returns an error, but the networking actually begins working. Go figure!

The name of the network profile, eth0 in my case, must match the name of the file you created when you copied the example configuration. You can see which profiles are available by:

```bash
> netctl list
```
Which gives me 'eth0' on my system. You might have different or additional ones.

If, on the other hand, you need to set your Raspberry Pi up with a static IP address, so that you know where to find it on your network all the time, proceed as follows:

```bash
> cd /etc/netctl
> cp examples/ethernet-static ./eth0
```
This time we need to edit the configuration file, `eth0`, and add in the settings we need. Before you proceed, you will need the following:

- Static IP address - I'm using 192.168.1.36. The netmask I'm using is 255.255.255.0 which is defined as '/24' or the first 24 bits.
- An Interface name - I'm using 'eth0'.
- A default gateway - I use 192.168.1.1.
- Some DNS names servers - I'm using 8.8.8.8 and 194.168.4.100 and 194.168.8.100 - which are Google and Virgin Media's DNS servers. You should use the ones you have been told to use by your own ISP.

When you have finished editing the file, it should look like the following, using my settings above:

```text
Description='Anything you like'
Interface=eth0
Connection=ethernet
IP=static
Address=('192.168.1.36/24')
Gateway=('192.168.1.1')
DNS=('8.8.8.8' '194.168.4.100' '194.168.8.100')
```

Save the file and exit the editor. Try starting the network:

```bash
> netctl start eth0
```

On my system, that always returns an error, but the networking actually begins working. Go figure!

{{< alert theme="info" >}}
It appears that the ethernet connection is running under control of the `ifplug` daemon. It will be started whenever a cable is inserted. Hmmm.

The file in question that controls what is started by ifplugd is `/etc/ifplugd/ifplugd.conf` - have a look and see if your "eth0" is listed in the `interfaces` list.
{{< /alert >}}

After this, I can ping 8.8.8.8 and also ping google.com and in both cases, get a correct response.

If your network starts ok, without error, and works, then you can configure it to always start on boot by running the following command:

```bash
> netctl enable eth0
```

If you ever decide to disable it again, it's equally as simple:

```bash
> netctl disable eth0
```

## Set Your Timezone

So far so good. We need to correctly set the timezone next. If it isn't already done so that is.

First we check the current setting:

```bash
> ls -l /etc/localtime
```

If there is no file named `/etc/localtime` then you need to set it up. If there is a file, but it comes back something like the following:

```text
... /etc/localtime -> /usr/share/zoneinfo/Europe/London
```

And you don't want to be in this timezone, then you need to change it. If the file already exists but is wrong, delete it:

```bash
> rm /etc/localtime
```

Now recreate the file by sym-linking to an existing timezone file:

```bash
> ln -s /usr/share/zoneinfo/Australia/Brisbane /etc/localtime
```

The example above, obviously, sets the timezone to be suitable for someone living in the Brisbane, Australia timezone.

## Set the Hostname

The Arch installation on your raspberry Pi comes with a strange default hostname - _alarmpi_. We should change it to something meaningful.

In the following, I've set my hostname to "raspberrypi" - what else? ;-)

```bash
> echo raspberrypi > /etc/hostname
```

The new hostname will take effect after a reboot. If you wish to set it for the current session then:

```bash
> hostname raspberrypi
```

Your shell session prompt will still be showing the old hostname, but if you run the command `'su -'` then the prompt will change to display the new hostname.

## Add a New Pi User

So far we have been working in the root user, but the time will come when we need to work in a "safe" user that doesn't have the ability to completely trash the entire system! We shall create a 'pi' user with a password, as follows:

```bash
> useradd -m -g users -s /bin/bash -G audio,games,lp,optical,power,scanner,storage,video pi
```

The above command adds the pi user to the system, with its main group being users. It uses the bash shell and is a member of a whole list of subordinate groups. it does not yet have a password, so, as before:

```bash
> passwd pi
```

And follow the prompts:

Enter new UNIX password: 
Retype new UNIX password:
passwd: password updated successfully

## Install Sudo

The new pi user cannot run root's privileged commands, until we install the sudo option. As root:

```bash
> pacman -S sudo
```

You will be prompted to confirm your wish and the `sudo` command will be installed.

Now we need to tell `sudo` that pi is allowed to use it. We will do this by adding the pi user to the sudo group. We still need the `/etc/sudoers` file updating though, so use the `visudo` command to edit it.

```bash
> visudo
```

Locate the lines that are currently commented out as follows:

```text
## Uncomment to allow members of group sudo to execute any command
# %sudo ALL=(ALL) NOPASSWD: ALL
```

Uncomment the second of the above lines, so that it reads as follows:

```text
## Uncomment to allow members of group sudo to execute any command
%sudo ALL=(ALL) NOPASSWD: ALL
```

Use the ESC key, followed by **:wq** to write the file and exit.

Now add a new group named sudo to the system and add the pi user to it:

```bash
> groupadd sudo
> usermod -a -G sudo pi
```

And we can check if it stuck, with the following command:

```bash
> groups pi
```

If all went well, you should see the full list of groups we set up when we created the pi user, and the new sudo group as well:

```text
lp games video audio optical storage scanner power sudo users
```

When the pi user next logs in, or starts a new shell, then it will be allowed to use the `sudo` command.

## Date and Time

If you execute the `date` command before setting up networking, you will most likely find that the date is way back in the 1970's. Not helpful at all.

However, once you get the internet working, you will find that the same command now returns the correct date and time. This is because there is an `NTP` daemon running in Arch that connects to a time server and sets your Pi's clock to the correct time.

As the Pi doesn't have a battery backed clock as standard, although you can add one, this means that the clock will be wrong each time you boot up until ntp has done its thing. Beware, especially if you are creating source code files that `make` is used to build - if the dates are out, then unnecessary compilations might be done.

## Reboot

That's it for this first instalment. A quick reboot and everything should be in order.

# reboot

You will note, however, that the system is still in console only mode. Want a GUI? Tune in for the [next exciting instalment](/posts/2013/06/beginners-guide-to-arch-linux-on-the-raspberry-pi-part-2/ "Beginner’s Guide to Arch Linux on the Raspberry Pi – Part 2")....

Have fun.
