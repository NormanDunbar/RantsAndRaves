---
title: "FLAC to MP3 as Easy as Pie!"
date: "2017-10-27"
categories: 
  - "linux"
  - "music"
---

I have ripped all my music, well most of it, to FLAC for the quality aspect. Sometimes though, I need to convert to MP3 for some of the lesser audio players out there that I might have to use from time to time.

I have recently come across a pretty nifty (Linux) way to do this, without having to cope with having duplicate files in FLAC and MP3 formats on my hard drives.

The utility I've discovered is called mp3fs and is a FUSE file system whereby a normal, non-root user, can `mount` the FLAC folder and see the contents as MP3 files.

Once mounted in this way, the MP3 files can be played by, or copied to, a _less well enabled_ device and will be converted to MP3 on the fly. I don't then have to have MP3 files clogging up my music hard drives.

### Installation

On my Linux Mint 18.2 setup, it's a simple one liner:

```bash
sudo apt-get install mp3fs
```

### Usage

First, create the folder where my FLAC files will appear as MP3 files. I'm calling mine `mp3`:

```bash
mkdir ./mp3
```

Then mount the FLAC folder on to the new `mp3` folder. The FLAC files live in `/media/norman/USB_MUSIC` and sub-folders below this mount point:

```bash
mp3fs -b 192 /media/norman/USB\_MUSIC ./mp3
```

The `-b 192` part sets the bit rate for the MP3 output files. Other values are available.

Now, if I do a quick check, I see the following:

```bash
ls ./mp3
Benny Andersson  Carole\_King  Fleetwood\_Mac  Tangerine Dream  Zero Project
```

```bash
ls ./mp3/Tangerine\\ Dream/
Quantum\_Gate
```

```bash
ls ./mp3/Tangerine\\ Dream/Quantum\_Gate/
01 - Sensing\_Elements.mp3
02 - Roll\_the\_Seven\_Twice.mp3
03 - Granular\_Blankets.mp3
04 - It\_is\_Time\_to\_Leave\_When\_Everyone\_is\_Dancing.mp3
05 - Identify\_Proven\_Matrix.mp3
06 - Non-Locality\_Destination.mp3
07 - Proton\_Bonfire.mp3
08 - Tear\_Down\_the\_Grey\_Skies.mp3
09 - Genesis\_of\_Precious\_Thoughts.mp3
```

It's looking good. Now I can copy my wife's new CDs from the folders above to the device she wants to play them on, as MP3 files. My FLAC ripped files will be converted to MP3 on the fly as the copy progresses.

Once completed, I can unmount the `mp3` folder as follows:

```bash
fusermount -u ./mp3
```

There are numerous options that can be supplied to the mp3fs command, including one to automatically unmount the folder after the file operation has completed. I prefer to manual unmount things as I _might_ want to do other stuff later.
