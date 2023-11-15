---
title: "Raspberry Pi, PiZero, Raspbian Jessie, Networking and WiFi Setup"
date: "2016-01-20"
categories: 
  - "gadgets-gizmos"
  - "linux"
  - "raspberry-pi"
---

With a title like that, I should get some hits! ;-)

**Note**: Updated 6th October 2018 to cover Raspbian Stretch which is slightly different to Jessie. But not much.

**Note**: Updated 8th February 2016 to cover the new Raspberry Pi 3 with built in WiFi and Bluetooth.

**Note**: Checked 20th October 2017 to make sure that the instructions below still apply, and work, on the new _stretch_ release of Raspbian. They do!

Since Jessie came along, networking have changed slightly, and there are numerous people having problems getting Jessie to connect to the internet, or just to WiFi. This might help. Oh, by the way, this is only a problem if the package named "raspberrypi-net-mods" has been installed - or so it seems.

If the above package _has_ been installed, at least it backs up your old config files in `/etc/network/*-old`. So you can revert, if necessary. However, we want to use the latest and greatest, so read on.

There are a couple of files that need to be updated:

### File /etc/network/interfaces (Jessie Only)

If you are on Raspbian Stretch, see the next section. This section applies to _Jessie_ only.

This file should resemble the following:

```text
auto lo
iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet manual

auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
wpa-conf /etc/wpa\_supplicant/wpa\_supplicant.conf
```

In case you are wondering, the "auto" lines mean that the Pi will attempt to bring up these networks at boot time. If you don't want this, remove the lines and use `sudo ifup eth0` or `sudo ifup wlan0` to bring them up manually.

It's probably best not to touch the "lo" device though, just in case! ;-)

### File /etc/network/interfaces (Stretch Only)

If you are on Raspbian Jessie, see the previous section. This section applies to _Stretch_ only.

This file should resemble the following:

```text
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d
source-directory /etc/network/interfaces.d
```

So, that's a little different to Jessie. What it means is that instead of filling the `interfaces` file with our settings, we need separate files for each device. (Ok, we could put them all in the same file, but keep different devices apart I say!)

Create the following files, and enter the text as shown, with whatever minor adjustments you need for your system.

#### /etc/network/interfaces.d/lo.conf

That's Ell, Oh.conf by the way, in lower case.

```text
auto lo
iface lo inet loopback
```

#### /etc/network/interfaces.d/eth0.conf

```text
auto eth0
allow-hotplug eth0
iface eth0 inet manual
```

#### /etc/network/interfaces.d/wlan0.conf

```text
auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
wpa-conf /etc/wpa\_supplicant/wpa\_supplicant.conf
```

In case you are wondering, the "auto" lines mean that the Pi will attempt to bring up these network devices at boot time. If you don't want this, remove the lines and use `sudo ifup eth0` or `sudo ifup wlan0` to bring them up manually.

### File /etc/wpa\_supplicant/wpa\_supplicant.conf

So far, so good. On both Jessie and Stretch, the following applies.

For a WiFi network, we need to update the `wpa_supplicant.conf` file.

Edit the file `/etc/wpa_supplicant/wpa_supplicant.conf` and make it look like the following example, similar that is, not identical.

```text
country=GB

ctrl\_interface=DIR=/var/run/wpa\_supplicant GROUP=netdev
update\_config=1

network={
   ssid="YourNetworkSSID"
   psk="YourVerySecretPassword"
#   scan\_ssid=1
#   priority=1
}
```
Ok, points to be very careful of:

- The `country=GB` is required from the 26/02/2016 Raspbian release. It probably does no harm to have it in with earlier versions, your mileage may vary of course. It's definitely required with the new Raspberry Pi 3 with built in WiFi and Bluetooth. If you are not sure of your country code, go to the Menu-> Preferences->Raspberry Pi Configuration utility and set it on the Localisation tab. You can pick from a list.

- There are _no spaces_ between names, equal signs and parameters etc. If you have any spaces, networking will not start. Ask me how I know!

- Lines with a '#' as the first character are comments. They are ignored.

- You need one `network={...}` entry for each WiFi network that you might use. Home, work, college, whatever. Each will have a different name in the `ssid` entry.

- The `scan_ssid` and `priority` entries are optional.

- `Priority` determines the order in which your Pi will connect if more than one of the networks is available. It's best to give each one a different priority in that case! If more than one networks have the same priority, then the signal strength and/or security policy may be used to choose the best one to connect to.

- `Scan_ssid` determines how the network will be scanned for. From the docs, and if you follow this, you are better than I am!
    
    > SSID scan technique; 0 (default) or 1. Technique 0 scans for the SSID using a broadcast Probe Request frame while 1 uses a directed Probe Request frame. Access points that cloak themselves by not broadcasting their SSID require technique 1, but beware that this scheme can cause scanning to take longer to complete.
    

I have two WiFi networks that the Pi can connect to, so I have two different entries in my configuration file.

So, replace `"YourNetworkSSID"` above, with the name of your WiFi network. If the network doesn't broadcast the SSID, then you need to know exactly what it is.

Replace `"YourVerySecretPassword"` with your own network's password.

Save the changes. My own file looks like this:

```text
country=GB

ctrl\_interface=DIR=/var/run/wpa\_supplicant GROUP=netdev
update\_config=1

network={
   ssid="HomeWIFI"
   psk="YoullNeverGuessThisPassword"
   priority=1
}

network={
   ssid="OfficeWIFI"
   psk="YoullNeverGuessThisPasswordEither"
   priority=2
}
```

I have a home and office Wifi in the same place. I work from home and can use either.

In the past, you would set up your static IP addresses, gateways, netmasks etc in the `/etc/network/interfaces` file. No longer! It is now held in the `/etc/dhcpcd.conf` file. (Which I find very difficult to (a) remember and (b) type!

### File /etc/dhcpcd.conf

Only do the following if you wish to assign a static IP address to your Pi when it's on wired (eth0) and/or WiFi (wlan0) networks. I have mine set to have a static IP on both, so my file looks similar (ok, identical) to the example below.

Edit the file `/etc/dhcpcd.conf` and search for the `interface eth0` or `interface wlan0` lines. They may not be present. If you don't find the entries, scroll to the end of the file.

Add the following configuration lines:

```text
# Static IP configuration for eth0.
interface eth0
static ip\_address=192.168.1.50/24
static routers=192.168.1.1
static domain\_name\_servers=8.8.4.4 8.8.8.8

# Static IP configuration for wlan0.
interface wlan0
static ip\_address=192.168.1.51/24
static routers=192.168.1.1
static domain\_name\_servers=8.8.4.4 8.8.8.8
``` 

Some more points to be very careful of:

- As mentioned above, you _only_ need to do this if you wish to allocate a static IP address to your Pi when it's on wired (eth0) and/or WiFi (wlan0) networks. If you don't care what IP it has, simply ignore this section.

- Again, there are _no spaces_ between names, equal signs and parameters etc. If you have any spaces, networking will not start. Ask me how I know!

- You need one `interface ...` entry for each `iface ...` that you have defined in `/etc/network/interfaces` - don't set one up for the "lo" device though! The names must match too.

- Yes, you do need to repeat the information for routers etc in each entry. They don't get shared, which is a shame. It would be nice to have a global section where I could define these once, then only define the changeable stuff in each `interface` entry. Still, c'est la vie, as they say in Wales.

Ok, the static ip\_address line defines that I'm using an IP address of 192.168.1.50 when on eth0, or wired network, and 192.168.51 when on WiFi. This allows me to use both, at the same time, if I wish, without an IP clash.

The `/24` means that the first 24 bits of the IP address are my netmask, or 255.255.255.0. This identifies my network as 192.168.1.xxx and my devices as 50 and 51, in this case.

My router is on address 192.168.1.1, so that's what goes into the routers entry.

I use Google's name servers for my DNS name resolution, they are usually always available, but you can add others if you wish. Your ISP might have some that you can use. Separate each one with a space.

By the way, _don't_ use a name here, always use an IP address. You can't resolve the names you use for the DNS servers if you don't have a DNS server to resolve them! (No, I didn't do that one myself, thanks for asking!)

Save the file.

That's should do it. A quick reboot is probably the best way to make sure everything is set up correctly, so insert a slight pause here while we reboot.

```bash
sudo reboot
```

When the system comes back up, run the following command to see what appears:

```bash
hostname -I  ## That's a capital 'eye' not a 'one'.
```

You should see the two static IP addresses you configured, if you did so, and if you have both eth0 and wlan0 up and running, like I do. If you are only connected to WiFi, for example, you should only see the one IP address for that.

If you see an extra IP address, then you probably still have some dhcp or static entries in your `/etc/network/interfaces` file, in the old style format, that need to be removed. Make sure that your file looks like mine above to get rid of the ghost IP address.

### And Finally...

[This might be a good article](/posts/2016/03/does-your-raspberry-pi-3-lose-wifi-connections-after-a-while/) to have a look at if you have problems with your WiFi connection seemingly _going to sleep_ which was known to be a problem with early raspberry Pi 3+ machines.



HTH

Cheers.
