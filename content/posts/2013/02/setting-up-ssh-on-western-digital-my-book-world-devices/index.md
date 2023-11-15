---
title: "Setting up SSH on Western Digital My Book World Devices"
date: "2013-02-11"
categories: 
  - "linux"
---

I have a network attached 1 Tb hard drive. It's a white book, Western Digital "My Book World" device. I use it for backups. I needed to set it up to allow the backup scripts ssh access without a password. Here's how I did it.

- The web interface was used in the normal way to create a user named _my_backups_.
- A share was set up automatically for this new user. The share name is the same as the user name - my_backups.
- using advanced mode->network, I ensured that NFS was enabled and that only devices on my internal network were able to access the shares. This is not necesary for the backups though, but it allows me to NFS mount the share as required.
- I logged onto the WD device using ssh, as root. I needed to edit the `/etc/passwd` file to move the home directory for my new user from `/shares` to `/shares/my_backups` otherwise, setttng up ssh affects (potentially) all users. And besides, I want my users to have their own home, not a shared one.
- The default permissions on the my_backups share is to allow group writes. Ssh _will not work_ if this is the case, so that has to go, as root:
    ```bash
    chmod g-w /shares/my_backups
    ```
- The ownership of the my_backups share is also dubious. So another quick change is called for, as root:
    ```bash
    chown my_backups /shares/my_backups
    ```
- Now I can login using ssh as the my_backups user, so logout as root, and back in again as my_backups.
- Now set up all the stuff (technical term!) required by ssh:
    ```bash
    cd
    pwd
    /shares/my_backups
    
    mkdir .ssh backups
    chmod 600 .ssh
    ls -al
    ...
    drw------- 2 my_backu jewab           6 Feb 11 15:56 .ssh
    drwx------ 2 my_backu jewab           6 Feb 11 16:27 backups
    
    cd .ssh
    touch authorized_keys
    chmod 700 authorized_keys
    ls -l
    -rwx------ 1 my_backu jewab           0 Feb 11 15:58 authorized_keys
    
    exit
    ```
    
- Now, I copy my ssh public key to the new `authorized_keys` file, from my laptop, which is the device I'm backing up up to the shared drive:
    ```bash
    $ cd ~/.ssh
    $ cat id_dsa.pub | ssh root@wd 'cat >> ~my_backups/.ssh/authorized_keys'
    root@wd's password: ******
    ```
    You will notice that I'm connecting as root to do this. For some reason, on the device, attempting to do this as my_backups gives me an error `sh: cannot create .ssh/authorized_keys: Permission denied` which is strange, as the file exists and is correctly owned by the my_backups user. However, there are a few foibles in the OS that is running on the device, so logging in as root works!
- Now I can login as my_backups without a password, from my laptop. And more to the point, my backup script can do it too.
    ```bash
    ssh my_backups@wd
    
    pwd
    /shares/my_backups
    ```
- Talking of backup scripts, it's nothing more than an `rsync` command. The rsync option `--archive` cannot be used as it won't let the `--times` part of it run and lots of errors are scrolled up the screen.
    
    To get around this, I'm using `--perms --delete --update --verbose --links --progress --recursive` instead. The destination on the rsync command is specified as `my_backups@wd:backups` which does the job nicely!
    
    > Ahem, of course, the first time I did this, I neglected to create the backups directory didn't I? So what happened on the first actual run? Rsync happily `--delete`d the .ssh directory holding my keys, so further testing didn't work and requested the password that I had worked so hard to get rid of!
