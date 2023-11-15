---
title: "Getting Arduino Working from a Windows 7 VirtualBox Guest"
date: "2015-09-25"
categories: 
  - "gadgets-gizmos"
  - "linux"
  - "virtualbox"
  - "windows"
---

Do I like problems or what? :-) I'm running Linux Mint 17.2 as my host, and I have a VirtualBox 5.0 VM running Windows 7 Professional. I decided I'd like to be able to run the Arduino software from within the VM, but not talking to an Arduino, but to a bare bones setup and programming AtTiny85 devices. The following might be of use to other people's needs as it explains how the FDTI device cane be automatically assigned to the VM rather than to the host, when plugged in and the VM is running.

### Set up the Host First

There are two things you need to do on a Linux host to enable the USB deviced for the guest(s). The first is to ensure that the user you normally run VirtualBox under is a member of the `vboxusers` group, if not, add the user and logout and back in again. (You don't need to reboot the machine, just logout of your user and back in again.

```bash
$ groups  
adm dialout cdrom sudo dip plugdev lpadmin sambashare

$ sudo usermod -a -G vboxusers your_user_name
```

Now logout and back in again.

```bash
$ groups
adm dialout cdrom sudo dip plugdev lpadmin sambashare vboxusers
```

We can see that the `vboxusers` is now active.

The second task that needs doing is to discover the FDTI device's vendor, product and serial numbers.

```bash
$ lsusb
...
Bus 007 Device 007: ID 10c4:ea60 Cygnal Integrated Products, Inc. CP210x UART Bridge / myAVR mySmartUSB light
...
```

The above is fine, the main `ID` field shows that the id of this particular device is `10c4:ea60` with is the vendor id and the product id, both of which should be unique. There is no serial number though, and that _might_ be needed if there are more than one revision of the device. Try this:

```bash
$ VBoxManage list usbhost
...
UUID:               9f963985-1fb7-49e0-b066-e1261a6a1985
VendorId:           0x10c4 (10C4)
ProductId:          0xea60 (EA60)
Revision:           1.0 (0100)
Port:               0
USB version/speed:  1/Full
Manufacturer:       Silicon Labs
Product:            CP2102 USB to UART Bridge Controller
SerialNumber:       0001
Address:            sysfs:/sys/devices/pci0000:00/0000:00:1d.1/usb7/7-1//device:/dev/vboxusb/007/007
Current State:      Captured
```

So, we have serial number 0001. We can now set up a VirtualBox USB filter to cause that particular FDTI device to be connected directly to the Windows 7 guest when it is running and the device is plugged in. Note that we must note down the vendor and product ids exactly as they are displayed, not the uppercased version in brackets after it!

### Create a VirtualBox USB Filter

The Windows VM needs to be closed, for best results. The FDTI device should also be plugged in.

- Run the VirtualBox manager and click on the Windows VM on the VM list on the left.
- Click settings to open the settings dialogue for that particular VM.
- Scroll down and select the USB setings on the left.
- On the right, make sure that USB 2.0 (EHCI) Controller is selected. I've found that setting the USB 3.0 controller fails with my laptop and the Windows guest complains that it has no USB drivers. I don't have USB 3 on my laptop, so 2 it is for me!
- Click the second button down on the far right, it's a USB plug with a '+' on it. This adds a new filter.
- Select the FDTI from the list that appears.
- Right-click it, and select edit. Make sure that product and vendor Ids, and the serial number match. The other details can be either left as is, or ignored. Be aware that anything in any of these fields must match the devices information _exactly_. OK back out of this dialogue. If no devices appear in the list, your user is not in the `vboxusers` group, or it is not yet active.
- Make sure that the filter is enabled by checking the checbox to the left of the name.
- Start the VM.

If the device is unplugged when the VM is subsequently started, it won't matter. With the filter in place, plugging the device into a USB slot will cause it to be assigned to the guest and not to the host.

### Install Arduino

In the Windows VM, open Firefox - other browsers are available. Internet Explorer is _not_ a browser ;-) - and navigate to [https://www.arduino.cc/en/Main/Software](https://www.arduino.cc/en/Main/Software) and download the software, 1.6.5 was the latest version at the time of writing. Make a donation, if you wish. When the software has downloaded, install it. It will attempt to install a few USB drivers and such like, make sure that you allow it to do so. Windows is so naggy!

You may have to reboot Windows to make the new drivers stick. Sigh.

Startup the Arduino IDE and configure it for your Arduino board (Tools->Board), Port (Tools->Port) and programmer (Tools->Programmer). Make a note of the port. Open the ubiquitous _blink_ sketch in the usual manner, and upload it. If it uploaded then well done, you are now able to program Arduinos from within a VM.

In my case, it failed as it couldn't open port COM1, it said that the port could not be found. Running device manager (start, search, type device, select device manager from the list) and scrolling down to COM and LPT ports showed that only LPT1 was present.

### Install a Proper Driver

Go directly to [http://www.silabs.com/products/mcu/Pages/USBtoUARTBridgeVCPDrivers.aspx](http://www.silabs.com/products/mcu/Pages/USBtoUARTBridgeVCPDrivers.aspx) where you can download a driver to make FDTI devices work on just about every version of Windows. Select the appropriate version for your _guest_ operating system. You will download a zip file.

Find the zip file, and extract it, then change into the folder that was created (CP210x_VCP_Windows\\CP210x_VCP_Windows in my case) and run either the 32bit (x86) or 64 bit installer depending on your guest. Mine was 64 bit, so I executed CP210xVCPInstaller_x64.exe.

Guess what? Another reboot is required. Windows sucks!

After the reboot, open the blink sketch again and go to tools->port, your com port should now appear in the list, mine was deemed to be COM3, so I chose that.

Click upload, and be amazed as your Arduino starts working!

### AtTiny85 Devices

Need to program these? Or the AtTiny25? Or AtTiny45? See how to set up Arduino IDE 1.6.x at [http://highlowtech.org/?p=1695](http://highlowtech.org/?p=1695) where all is explained. It just works.

If anyone is interested, I'm connecting my FDTI device to an Arduino Uno lookalike (actually, it's a Shrimp, see [https://start.shrimping.it/kit/stripboard.html](https://start.shrimping.it/kit/stripboard.html), built on a breadboard style strip board) with an AtMega328 running the ArduinoISP program. My programmer in the IDE is _Arduino as ISP_ and **not** ArduinoISP - confusing or what - I've installed the AtTiny stuff as per the link above. It all just works.

Oh, and one other thing, the AtTiny25/45/85 has PWM on physical pin 3; PB4/OC1B, as well as on physical pins 5 and 6; PB0/OC0A and PB1/OC0B/OC1A. The diagram on the above link isn't quite correct, but I've informed them of this.

Have fun.
