---
title: "Messing About with Samba Shares on my Volumio Server"
date: 2023-04-15T14:47:02+01:00
lastmod: 2023-04-15T14:47:02+01:00
description: "How to mount SAMBA shares from the Volumio server, on other devices."
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
    - "SAMBA"
---

As you might know, if you have read my two previous posts, I have a [Raspberry Pi](https://www.raspberrypi.com/ "https://www.raspberrypi.com/") set up as a Music *player* using [Volumio](https://volumio.com "https://volumio.com") and while this is excellent when I'm sat at my laptop -- probably writing another Arduino book! -- I can't play my music from my tablet or phone while elsewhere in the house. I still want to install [MStream](https://mstream.io "https://mstream.io") so I'm no setting up a second Raspberry Pi to install MStream on, ad hopefully, get it to read all the music from the Volumio server.


I have already installed [Bubble UPNP](https://bubbleupnp.en.uptodown.com/android "https://bubbleupnp.en.uptodown.com/android"), from the Google Play Store, onto my phone and tablet. This app comes highly recommended. Unfortunately, in the free version, playlists are limited to 16 tracks only, and many of my albums have more tracks that this. So, unless I upgrade and pay (!) or mess around with playlists consisting of the first 16 tracks then the remainder, I'm not sure this is the best option for me, especially when the MStream Android app is so good -- I've been running it connected to their own demo server, and it's brilliant!

## Useful SAMBA Links

I found the information on the following links to be extremely helpful in setting up my SAMBA shares.

* [Configuring Samba](https://linuxconfig.org/how-to-configure-samba-server-share-on-ubuntu-20-04-focal-fossa-linux "https://linuxconfig.org/how-to-configure-samba-server-share-on-ubuntu-20-04-focal-fossa-linux")
* [Mounting SAMBA shares at boot time](https://linuxconfig.org/how-to-mount-a-samba-shared-directory-at-boot "https://linuxconfig.org/how-to-mount-a-samba-shared-directory-at-boot")


## Configuring the Volumio Server

The first task is to get onto the Volumio server to set up a new, read only, share that other devices can mount and see all the USB attached thumb drives I store my music on. In theory, this should also allow them to see any new drives that I add. 

To configure SAMBA, I need to be root, but I'm logged in as volumio, no worries:

```bash
sudo -i
cs /etc/samba
```

Now, create a backup before editing the configuration file, `smb.conf`:

```bash
cp smb.conf smb.conf.backup.ND
nano smb.conf
```

In the editor, we can see that there are three SAMBA mounts defined:

* Internal Storage on `/data/internal`, we are not interested in this one.
* NAS on `/mnt/NAS`, again, we are not interested.
* USB on `/mnt/USB`, and this one, we are most definitely interested in!

These are configured as read-write, but I don't want to allow everyone and their dog to write all over my music collection, so I'll be setting up a new share which will be read only. At the bottom of the file, add the following text:

```text
[MSTREAM]
        # This allows anonymous (guest) access
        # without authentication. 
        # Only USB mounts are accessible, read only access.
        path = /mnt/USB
        read only = yes
        guest ok = yes
        guest only = yes
```

After saving the file with `CTRL-X, Y, ENTER` we need to restart the SAMBA daemon, `smbd`:

```bash
systemctl restart smbd
```

## Testing the New SAMBA Share

On a different device -- laptop or Raspberry Pi -- I can test if the new share is available as follows.

First, we need our own user's user and group ids:

```bash
id

uid=1000(norman) gid=1000(norman) groups=.....
```

You will need to substitute your own user and group ids as necessary. Now we can run the test:


```bash
sudo -i    ## If necessary, we need to be root.
mkdir -p /mnt/mstream
mount -t cifs  -o guest -o uid=1000 -o gid=1000 //192.168.1.41/MSTREAM /mnt/mstream
```


Assuming that the `mount` command, above, comes back with no errors, then we should be able to see my mounted USB devices on the remote Volumio server:

```bash
ls mnt/mstream

0959-0DF7  30E5-1614
```

And, check within one of them:

```bash
ls mnt/mstream/0959-0DF7

'Alice Cooper'                    Journey
'Amy MacDonald'                  'Kate Bush'
'Anderson Bruford Wakeman Howe'  'Katie Melua'
'Andrea Bocelli'                 'Marillion'
...                              ...
```

So, it all seems to be working. Unmount the test and, if necessary, remove the mount point if it is no longer needed. In my case, it will hopefully be used when I install MStream, so I'm leaving it, for now at least. We should still be logged into root through the `sudo -i` command earlier.

```bash
sudo umount /mnt/mstream
sudo rmdir /mnt/mstream    ## If mount is test only and no longer required.
```

## Configure the Client Device

We would like the client device, where MStream is to be installed, to mount the SAMBA share at boot time. To do this we need to edit `/etc/fstab` on the client machine:

```bash
sudo vi /etc/fstab
```

Add a single line of text to the file, at the end is probably best:

```text
//192.168.1.41/MSTREAM /mnt/mstream cifs uid=1000,gid=1000,guest 0 0
```

We can test this with:

```bash
sudo mount -a
ls mnt/mstream

0959-0DF7  30E5-1614
```



