---
title: "Give Your Raspberry Pi Turbo Mode"
date: "2012-09-19"
categories: 
  - "gadgets-gizmos"
  - "raspberry-pi"
---

And while you are at it, no warranty problems when you do so!

An announcement on the Raspberry Pi Foundation web site, [read it here](http://www.raspberrypi.org/archives/2008 "http://www.raspberrypi.org/archives/2008"), (**Update: 23/02/2023**: Sorry, dead link now.) explains that it is now possible to use Turbo mode on your Raspberry Pi without invalidating your warranty by over volting the device.

Please go and [read it](http://www.raspberrypi.org/archives/2008 "http://www.raspberrypi.org/archives/2008") and then pop back here for instructions on adding Turbo mode to your own Raspberry Pi.

Back again? Ready to begin? Let's do it!

The following assumes that you are running Raspbian as your chosen distro, and that you are logged in as the `pi` user.

```bash
sudo apt-get update
sudo apt-get upgrade
sudo raspi-config

### Select the "update" option which updates raspi-config and exits.
```

```bash
sudo raspi-config
    select the new "configure overclocking" option and choose one.
    select finish.
    select yes to reboot now.
```

On reboot, your Raspberry Pi will be running \[much\] faster _under load_ and, if it gets too hot, will slip back to normal non-turbo mode automagically.

It is possible that your raspberry Pi will not boot if the settings are too high for your system. If this is the case, hold down the `shift` key during the boot.

I set mine to full TURBO mode and it booted, but the CPUFreq Monitor (see below) shows 700 MHz rather than 1 GHz, however, this is because the Pi is now running in automatic gearbox mode - when the CPU usage gets above 85%, then it switches to a higher gear.

You can see this if you have the CPUFreq Monitor installed (see below) and, in a shell session run something like `sudo apt-get update` and while that is running, hover over the CPUFreq Monitor icon to see the speed changing up and down as the load comes and goes.

You can see the CPU Frequency in use in a console as per the following example:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling\_cur\_freq
1000000
```

The output from which will be the frequency running now, in KHz. Divide by 1,000 to get the MHz frequency. The example above was taken while an `sudo apt-get update` was in progress. After the hard work was done, it dropped again:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling\_cur\_freq
700000
```

As mentioned above, if your Pi refuses to boot up with your chosen settings, simply reboot while holding down the `shift` key to skip the turbo mode setting, then `sudo raspi-config` to set it to a lower setting again.

You can, if you are running a GUI using XFCE, add the temperature sensor output and the current CPU frequency settings to your task bar, as follows:

- Right-click the task bar and select "Panel settings"
- Click on the "Panel Applets" tab
- Click the "Add" button.
- Select "Temperature Monitor" from the list, and click ADD.
- Use the "Up" and "Down" buttons to put it where you want it to be.
- Click the "Add" button.
- Select "CPUFreq Frontend" from the list and click ADD.
- Use the "Up" and "Down" buttons to put it where you want it to be.
- Click the "Close" button.

You should see a couple of new items in the task bar. One is a two digit number (or hopefully two digits) in green text, and the other looks like a, well, I'm not sure what it looks like in my VNC session!

Anyway, hover the mouse over each and you will get told what's what. The CPUFreq Monitor tells you what frequency you are running at and the temperature gauge stays green as long as the Raspberry Pi is running within temperature bounds. If the text turns red, it's getting too hot and your Pi should shut down to a slower overclock/overvolt to reduce the temperature.

You can configure these applets by right-clicking on them, and selecting the applet's own settings menu option (at the top of the menu).

Again, there is a console tool to allow you to see the current temperature:

```bash
cat /sys/class/thermal/thermal\_zone0/temp
42236
```

The temperature reported in in thousandths of a degree Celsius. Divide by 1,000 to get degrees C. (Add 273 to degrees C to get degrees Kelvin!)

See the original article, links above, for full details of what else is included in this release. You can also [read this article](http://www.raspberrypi.org/phpBB3/viewtopic.php?f=66&t=17788&p=176847 "http://www.raspberrypi.org/phpBB3/viewtopic.php?f=66&t=17788&p=176847") on setting up the other options in the new release.
