---
title: "Streaming Music from my Volumio Server"
date: 2023-04-14T15:17:02+01:00
lastmod: 2023-04-14T15:17:02+01:00
description: "How to stream music from my Volumio server to my other devices."
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
    - "Music"
    - "Streaming Music"
---

I have a [Raspberry Pi](https://www.raspberrypi.com/ "https://www.raspberrypi.com/") set up as a Music *player* using [Volumio](https://volumio.com "https://volumio.com") and while this is excellent when I'm sat at my laptop -- probably writing another Arduino book! -- I can't play my music from my tablet ot phone while elsewhere in the house. My original plan was to install [MStream](https://mstream.io "https://mstream.io") but thanks to the way that Volumio takes over the Pi, I was unable to install the MStream dependencies, *nodejs* and *npm* to that was a bummer straight away. However, this turned out to be quite simple to fix, although I had to kisss MStream goodbye at this point.

I first installed [Bubble UPNP](https://bubbleupnp.en.uptodown.com/android "https://bubbleupnp.en.uptodown.com/android"), from the Google Play Store, onto my phone and tablet. This app comes highly recommended. There is a free -- with ads -- version and a £3.99 pay for version. Being Scottish and living in Yorkshire, I have two reputations for meanness to live down to, so I went for the free version. This is especially handy as I'm using [Pi Hole](https://pi-hole.net "https://pi-hole.net") to block ads anyway.

After installing BubBle UPNP, I opened it, and opened the *Local and Cloud* option and set up a pair of servers, *Volumio One* and *Volumio Two* -- meaningful names eh? These were configured as *SMB* (Samba) server types and mapped to the two large USB thumb drives I'm using on the Raspberry Pi to hold my music.

Now I can stream music from my Volumio server to my devices. Easy! 

I could have just set up a single Volumio server and left the optional folder blank, but then I'd have had to navigate to USB then to one or other of the devices -- and they have very unmeaningful names -- so I preferred to do it as described above. I simply open one or other server and pick a folder to play. Job done.

