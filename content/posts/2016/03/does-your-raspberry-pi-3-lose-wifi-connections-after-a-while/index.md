---
title: "Does Your Raspberry Pi 3 Lose WiFi Connections After a While?"
date: "2016-03-15"
categories: 
  - "gadgets-gizmos"
  - "raspberry-pi"
---

If you find that your Raspberry Pi -- those with built in WiFi and Bluetooth -- loses the WiFi connection after a period of inactivity, then [this thread on the Raspberry Pi Forums](https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=138137), which will open in a new tab, might be of interest. Have a read. If you want to miss out on the preliminaries of the thread, [start reading here](https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=138137&p=929284#p920113) instead.

The problem is likely to be the power saving mode of the Wifi. To check the current setting:

```bash
sudo iw dev wlan0 get power_save
```

If you are/will be affected, it will reply with:

```
power_save: on
```

If so, all you need to do is:

```bash
sudo iw dev wlan0 set power_save off
```

I'm not affected yet, but I'm making a note here, just in case!

### Raspberry Pi Zero 2W

I received a new Raspberry Pi Zero 2W as a gift from the MagPi Magazine. I decided to upgrade the Zero W that was running my puppy monitoring camera. Everything was fine running Buster (I simply swapped over the microSD card) except power\_save mode was constantly turned back on after a reboot on the Zero 2W, but not on the Zero W. Weird!

This caused the Zero 2W to drop off the network and sometimes pinging it constantly would kick it back on, but other times a reboot was required (it runs headless).

Adding the following line to `/etc/network/interfaces.d/wlan0.conf` solved the problem.

```
wireless-power off
```

This appears to have worked. At least, after a reboot the power saving mode is disabled. My config file looks like this now:

```
auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
wireless-power off
```
