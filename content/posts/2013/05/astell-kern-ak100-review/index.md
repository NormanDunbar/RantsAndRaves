---
title: "Astell & Kern - AK100 Review"
date: "2013-05-23"
lastmod: "2023-02-23T14:22:23Z"
categories: 
  - "gadgets-gizmos"
---

{{< img src="/images/ak100.jpeg" title="Astell & Kern AK100"  width="179px" position="center" >}}


My wife bought me one of these because my old iRiver H340 has begun eating batteries. It's on its third replacement now, plus I have more music than fits the H340's hard drive. I do not buy Apple products, so any iWhatever (other than iRiver) was out of the question. Nothing personal, I just don't like them.

Finding a suitable replacement music player with decent storage, no hard drive, and Linux compatibility (64 bit essential) was very difficult especially if you are not fond of Apple kit. (Have I mentioned that I'm not?)

The package contains the device, a nice little fabric pouch, a demo micro SD card (2 Gb) with 5 tracks on it, a screen protector for the touch screen on front and one for the back, and a charging lead. There is no charger - you can use your PC's USB slot, your Kindle charger, your phone charger or whatever you have that is fitted with a standard USB slot to plug the lead into.

You must supply your own (decent quality) [earbuds with great sound quality](https://musiccritic.com/equipment/headphones/best-sound-quality-earbuds/) - or use the optical output to feed into a quality Hi-Fi system. The headphone socket can also supply line-out levels, so can be fed directly into an amp.

The manual is in various languages, and lives on the device itself. Save space by deleting the one(s) you don't need. Or all of them. Don't try and print it off on A4 paper, the pages are tiny - as is the print. Use a PDF reader to view the pages.

There is a copy of "iRiver Plus 4" for Windows on the device. Delete it. (See below!) and download the latest from iRiver.com. If you have a 32 bit Windows computer that is!

The supplied software (iRiver Plus 4) will not work on Windows 7 Professional 64 bit. It is 32 bit only. On 64 bit, it will not recognise that the device is plugged in. However, the software is not actually very good, so you are better off just using Windows Explorer and mounting the device as a USB drive. That works.

The two micro USB cards are valid up to 32 Gb but with the latest software and careful formatting, 64 Gb micro cards can be used giving a huge 160 Gb of storage. At least, that's what I'm told, I haven't got any 64Gb cards to try......yet!

Linux users, don't worry. The device, and cards is recognised as USB storage. Simply drag and drop files to the (up to three) drives. Works just fine. The manual says that only Windows and Mac (32 bit) are supported by iRiver Plus 4 and even then, only the card in slot 1 is recognised. Go figure.

The supplied iRiver Plus 4 software doesn't work under Wine on a 64 bit Linux system. It doesn't work under Windows 64bit, so there's not much chance of it working under Wine on a 64 bit system. 32 bit users may find that it does work under Wine. (Good Luck!)

Updating the firmware without iRiver Plus 4 is simple:

- Download the zip file from [http://www.iriver.com/support/download\_list.asp](http://www.iriver.com/support/download_list.asp "http://www.iriver.com/support/download_list.asp"). (Release 1.33 is available at the time of writing). (*Update 23/02/2023: This link is now a little bit useless!*)
- Unzip it to give you a hex file.
- Copy the hex file to the root of the main drive of the device - the same location where you see Music, Manual, AlbumArt and System directories.
- Unmount the USB drive that the device is pretending to be. Windows users, this means "safely remove". Don't just pull the plug!
- It will notice the new hex file and will install it. After a power off and back on, which it does automagically, you are now running the latest release.

Make sure you have a fully charged battery though, you do not want to expire half way through the upgrade. That would make an expensive brick!

Sound quality is excellent, I've heard stuff on my ripped CDs that I've never heard before. The device plays all sorts of formats, but the best ones are FLAC, WMA and OGG. Take my advice, don't even think about putting MP3 on this device - why would you bother, unless you really need the space savings.

The AK100 is solidly built. The chassis is a solid billet of aluminium which gives the device a decent weight. It is certainly not at all plasticy - and at the price, it shouldn't be!

Don't so as I did - spend ages trying to get the supplied screen protector on the front and the back case protector on the back. There is one already fitted on each side, so the ones you get in the package are spares! However, if you do do as I did, the touch screen works perfectly well through two protectors!

Controls are minimal. On the top is the power switch, input and output sockets.

The right side has only the volume knob. When playing, turning that increases or decreases the volume - but, if the screen is on, it shows the volume screen on the device and you can easily adjust the volume using the touch screen. Just drag the pointer where you want it to be.

The left side has three tiny buttons. Rewind, play, forward. Each has two functions as per the manual.

The bottom has the micro USB socket and the two slots for the micro USB cards under a heavy duty slide to open cover.

{{< alert theme="warning">}}
BEWARE! The manual shows only 1 micro card being inserted and shows how to align it. The bottom slot matches the diagram in the manual. However, the top slot requires the card to be inverted (turned upside down!) and put in the other way round.
{{< /alert >}}

So, in the bottom slot it's as per the manual. In the top slot, turn the card over so that the contacts are showing, and put it in that way. Be very very careful, the slots are very small, and putting the cards in the wrong way round might damage things and prevent you using them.

Also, the cards have to be pushed in until they click, and the same to get them out. Use another card to do this, because it's quite fiddly to get them in with fingers like mine. Ladies with decent nails will have no problem!

Building play lists on the device itself is interesting. You have to be playing a track to add it. It's not impossible, just fiddly. The supplied software is apparently better to use, but as it doesn't work on my system, I'm a tad up that creek! However, I'm a programmer and I'll be writing my own sometime soon.

{{< alert theme="success">}}
Actually, I've determined that the playlist is nothing more than a list of the paths to the files I want in the playlist. So, building one in a text editor is relatively simple. I'm still going to build a proper cross platform playlist builder though!
{{< /alert >}}

I loaded up the internal memory to almost full with a pile of FLACs, and on turning the device back on, the Auto Scan Database Rebuild told me that there wasn't enough room to rescan the tracks to put into the database. This is interesting.

I moved a pile of directories and files onto one of the SD cards and still got the same error, however, moving a few more directories resolved the problem. I think it's caused by the scan attempting to build the entire database in memory and _then_ write it to the file - or something like that - and it's unable to do it with all the tracks I have loaded.

Until I got the scan to work, I couldn't select tracks by Album, Artists, Genre etc, only by Folder. It's not a huge problem, my folders are arranged quite nicely by Artist then Albums beneath that and finally, the tracks.

I don't do album art, and because I have the screen configured to turn off after 10 seconds anyway, I really can't see the point of Album Art while a track is playing. I wonder if playing the same track on repeat for long enough will cause screen burn after a while? Plus, of course, it hogs valuable space for more music! Mind you, some music formats these days embed the art in the track - so if you were to purchase three tracks from Amazon, for example, you'd most likely end up with three duplicates of the same album art, embedded in your MP3s. Wonder if the art can be removed from the tracks. Hmmm.

The equaliser function is probably best left turned off. No matter what I did with it, I found the default sound settings better then anything I chose using the equaliser. It updates the sound in real time, when you stop moving your finger on the screen for a second, the sound will change. Keep moving your finger and it will patiently wait for you before changing anything sound wise.

The pouch supplied is good for protecting the device from scratches, and such like, but it is quite light and won't take too much stress. It is certainly not suitable for attaching to a belt for portability. This device is better off locked in an inside pocket or similar. I will be on the lookout for a small sturdy case to attach this to my belt.

Talking of which, if you do want to go portable with this player, and why not, get a headphone that comes out at 90 degrees rather than a straight in plug. It makes better sense, especially if you ever bend over while wearing it around your waist. The sticky out plug gets bent over by your bending body and could knacker the socket and/or headphone plug. My Sony active noise cancelling headphones have a 90 degree plug and work great.

So, in summary:

- It's excellent, a little pricey, but it's not Apple.
- Quality of build and sound is excellent.
- It handles optical in and out, if you have it.
- The software is described on a Hi-Fi forum as a "steaming POS" - it is. It doesn't work at all on 64 bit Windows systems, but to be fair, the manual does say that it must be 32 bit. It won't run on Linux, but under Linux, you don't actually need it.
- It doesn't play video, or allow you to read text or word files, it's a quality music player! If you want the other stuff, buy a tablet or a smart phone!
- Without the software, creating playlists is fiddly.
- The database rebuild scanner cannot cope with a huge amount of files all at once.
- I love it!
- **Added 23/02/2023**: There are no more software updates. Boo! Hiss!

And finally, I used to get around 4-6 hours playback from a fully charged battery on my H340. With this AK100, I'm past that already and I still show a full charge! I'm still on the very first (5 hour) charge.

Do you think I've been going on too long? ;-)

I did have a problem with the AK100 after a few weeks of almost constant use. The volume knob on the side became intermittent. I contacted my dealer - [Sound Fidelity](http://www.soundfidelity.co.uk "http://www.soundfidelity.co.uk") - and they were extremely helpful and had the unit returned for investigation. When it was returned to me, I noticed that the iRiver Service Centre had updated the software to 1.33 and the knob feels a lot more substantial than before in operation.

I tried the bluetooth function on the device and it hung! I had to do a power off reset (press the ^ button on the side, and the power button on top for 7 seconds) which wiped my settings but not the music or the database. Trying bluetooth again resulted in another hang and another reset. Maybe something in 1.33 is a problem?

~~Time to contact iRiver support~~.


**Update 23/02/2023**: Well, here we are in 2023. I've been using this device almost daily since 2013 -- 10 years! The AK100 is still working just fine. I've worn out another volumn control knob but it still works, just a bit intermittently. Luckily I can adjust the volume on the touch screen.

Bluetooth never worked. I suspect I've got a borked device, but rather than pay more good money to send it back -- out of warranty of course -- I just bought a dongle to plug into the headphone socket and used that for bluetooth. Works great with my Bose QC32 noise cancelling headphones. A gift/legacy from [my late mother,](/posts/2015/08/cheerio-and-rip-mum) who sadly died in 2015.


Cheers, Norm.
