---
title: "Raspberry Pi Sound"
date: "2012-10-12"
categories: 
  - "raspberry-pi"
---

Want to know how to redirect the sound from your Pi to either the HDMI or to the headphone socket? Read on ...

## Update - 11 February 2013

I've written a small command line utility - PiSound - to control the settings of the audio device on your RaspberryPi. You can download it from [https://github.com/NormanDunbar/PiSound](https://github.com/NormanDunbar/PiSound "PiSound on GitHub"). Enjoy.

## Deciding on the Output Device

The following command is all you need:

```bash
sudo amixer cset numid=3 n
```

Where the final 'n' is as follows:

- 0 = Auto - if HDMI is connected, use that, otherwise try the headphone socket.

- 1 = Sound goes to the headphone socket.

- 2 = Sound goes to HDMI socket.

> If your current user is a member of the audio group, the sudo parts of the `amixer` commands is not required.

You can make sure that it works by running a command such as:

```bash
aplay /usr/share/sounds/alsa/Front_Left.wav
```

You should be able to hear the sound if you have set the correct output as above.

Have fun.

## Raspbain 16th December 2012 - Update

It appears that something (a technical term) has gone wrong in the 16/12/2012 Raspbian release and/or after `sudo apt-get update && sudo apt-get upgrade` - which stops sound working.

You can tell if you are affected as follows:

```bash
$ sudo amixer controls

numid=4,iface=MIXER,name='Master Playback Switch'
numid=3,iface=MIXER,name='Master Playback Volume'
numid=2,iface=MIXER,name='Capture Switch'
numid=1,iface=MIXER,name='Capture Volume'
```

If you see the above, then you are affected and nothing you can do will allow you to redirect sound to the headphone socket. When I say _nothing_, I mean, nothing, except the following, as [explained here]( https://github.com/raspberrypi/firmware/issues/139 " https://github.com/raspberrypi/firmware/issues/139"), however, read the next section before you start removing stuff that you might need!

```bash
sudo apt-get purge --yes pulseaudio
...
sudo reboot
```

When your Pi comes back up, login and try again:

```bash
$ sudo amixer controls

numid=3,iface=MIXER,name='PCM Playback Route'
numid=2,iface=MIXER,name='PCM Playback Switch'
numid=1,iface=MIXER,name='PCM Playback Volume'
```

Now you can use the `sudo amixer cset numid=3 1` command as described above, to direct the audio output to your headphones.

## But I need PulseAudio ...

You might be in a situation where you need to keep PulseAudio installed. What to do? The answer is simple, in all the calls to `amixer` add in a card selector such as `-c 0`.

Normally, the PCM card is card 0 (zero) and PulseAudio is card 1 (one). Somehow, PulseAudio sets itself as the default card. _I haven't bothered to discover how or where it does this yet, I deinstalled PulseAudio on my system._

```bash
$ sudo amixer -c 0 controls

numid=3,iface=MIXER,name='PCM Playback Route'
numid=2,iface=MIXER,name='PCM Playback Switch'
numid=1,iface=MIXER,name='PCM Playback Volume'
```

Hooray! If the above works for you, where leaving out the card select options `-c 0` does not, then you must add `-c 0` to all the `amixer` commands below.

## Muting Sound

Numid=2 determines if sound is muted or not. To mute sound, regardless of its output device, do this:

```bash
$ sudo amixer cset numid=2 0
```

and to unmute the sound:

```bash
$ sudo amixer cset numid=2 1
```

## Volume Control

Numid=1 allows you to set the volume. The range is slightly strange in that it runs from -10239 to +400 with +400 being the maximum. On my system, I have a pair of X-mini powered and amplified speakers attached. A minimum value of -1000 gives a quiet sound, 0 (zero) gives reasonable sound and 400 is a bit too loud.

You adjust the volume as follows:

```bash
$ sudo amixer cset numid=1  -- -1000
```

Please note the double hyphen. This is required in front of any parameter that has a leading hyphen. In this case, the volume setting I require is -1000, so the double hyphen says "the following is a value, even though it has a hypen, it is not another flag or option!"

You can use the double hyphen in front of positive numbers as well, without any adverse effects.

```bash
$ sudo amixer cset numid=1  -- 234
```

Positive values between 0 and 400 appear unchanged while negative values between -1 and -10239 are rounded up to 0 to -10238.

The only way to get -10239 is to mute the sound.

Of course, being human, it would be nice to set the volume to something easily figured out, like a percentage, wouldn't it? This would be nice, for example:

```bash
$ sudo amixer cset numid=1 60%
```

No need to work out numbers in a weird range, no need for the double hyphens etc. Try it, _it works_! The range is obviously from 0% to 100%, anything outside of those boundaries will be limited to the appropriate percentage. Setting the volume to 0% effectively mutes the audio output.

## What are my Settings?

You may, if you wish, view all the settings on your Pi with the following single command:

```bash
$ sudo amixer contents

numid=3,iface=MIXER,name='PCM Playback Route'
  ; type=INTEGER,access=rw------,values=1,min=0,max=2,step=1
  : values=1
numid=2,iface=MIXER,name='PCM Playback Switch'
  ; type=BOOLEAN,access=rw------,values=1
  : values=on
numid=1,iface=MIXER,name='PCM Playback Volume'
  ; type=INTEGER,access=rw---R--,values=1,min=-10239,max=400,step=0
  : values=-1000
  | dBscale-min=-102.39dB,step=0.01dB,mute=1
```

If you wish to find the settings for one control only, use the same numid as you used to `cset` the control, but read the setting with the `cget` command instead:

```bash
$ sudo amixer cget numid=3

numid=3,iface=MIXER,name='PCM Playback Route'
  ; type=INTEGER,access=rw------,values=1,min=0,max=2,step=1
  : values=1
scale-min=-102.39dB,step=0.01dB,mute=1
```

There doesn't appear to be a way of fetching the current setting into a variable for use in, say, a bash script. Not unless you parse the data out of the returned string. The following python code will do this for you:

```python
import os
...
volume = None
stdout = os.popen('amixer cget numid=1')
try:
    volume = stdout.read()
finally:
    stdout.close()

# At this point, volume (a string) and contains all the output from
# the amixer cget command. Extract the volume value, if no exceptions
# occurred. It is None if there was an exception

if volume:
    volume = volume.split(':')[1].split('\n')[0].split('=')[1]

# At this point, volume contains the volume setting as a string.
...
```

There isn't, as far as I can find, any way of getting the current volume as a percentage. If that's what you want or need, I'm afraid you will have to work it out - as a slight clue, the following python code might help:

```python
...
# volume is a string holding the volume setting or is None. 
# We want it as an integer percentage, or -1 for errors..

if volume:
    percentage = int(((float(volume) + 10240) / 10640) * 100)
else:
    percentage = -1
...
```

You must `float` the string value or some calculations end up as zero percent due to the division by 10640.
