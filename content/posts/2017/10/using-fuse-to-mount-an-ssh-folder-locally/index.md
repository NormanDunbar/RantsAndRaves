---
title: "Using FUSE to Mount an SSH Folder Locally"
date: "2017-10-27"
categories: 
  - "linux"
---

I have recently come across a pretty nifty Linux utility that allows me to mount a remote filesystem on an SSH server, locally and without requiring root privileges to do so. The remote filesystem happens to be where my backups are located, so that's going to be useful for making and restoring backups!

The utility I've discovered is called `sshfs` and is a FUSE file system whereby a normal, non-root user, can `mount` the remote folder and see the contents as if they were actually in a local folder.

Once mounted in this way, the remote files can be copied to, from, deleted etc in the normal manner.

### Installation

On my Linux Mint 18.2 setup, it's a simple one liner:

```bash
sudo apt-get install sshfs
```

### Usage

First, create the folder where my remote files will appear. I'm calling mine `sshfiles`:

```bash
mkdir ./sshfiles
```

Then mount the remote folder on to the new `sshfiles` folder. The backup files live in the `norman/backups` folder, on a server named `wd` and the user account I need to login to is my own, `norman`:

```bash
sshfs norman@wd:norman/backups ./sshfiles
```

Now, if I do a quick check, I see the following:

```bash
ls ./sshfiles

Backup\_scripts  Downloads       Records         data            
Calibre         Home            SourceCode              
```

It's looking good. Now I can copy my local folders to the backup device by copying them _locally_ to the `sshfiles` folder. The `sshfs` utility will do the needful in copying them across the network to the correct server.

Once my backups (or restores) are completed, I can unmount the `sshfiles` folder as follows:

```bash
fusermount -u ./sshfiles
```

