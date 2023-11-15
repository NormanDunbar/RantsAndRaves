---
title: "Tnsnames.ora, IFILE and Network Drives on Windows"
date: "2016-05-19"
categories: 
  - "oracle"
  - "windows"
---

I've recently begun a new contract migrating a Solaris 9i database to Oracle 11gR2 on Windows, in the Azure cloud. I hate windows with a vengeance and this hasn't made me change my opinion!

One of the planned improvements is to have everyone using a standard, central `tnsnames.ora` file for alias resolution. A good plan, and the company has incorporated my own [tnsnames checker utility](http://qdosmsq.dunbar-it.co.uk/blog/2014/12/tnsnames-checker-utility/) to ensure that any edits are valid and don't break anything.

I found that the `tnsnames.ora` in my local Oracle Client install, was not working. Here's what I had to do to fix it.

In my local `tnsnames.ora`, I had something like the following:

```
IFILE="\\servername\share_name\central_tnsnames\tnsnames.ora"
```

_(Server names etc have been obfuscated to protect the innocent!)_

However, using the above caused `tnsping` commands, or connection attempts to time out or simply fail:

```
tnsping barney

TNS Ping Utility for 64-bit Windows: Version 11.2.0.1.0 - Production on 19-MAY-2
016 12:16:42

...

TNS-03505: Failed to resolve name
```

If the standard `tnsnames.ora` file was copied locally, and `IFILE`'d, then it all just worked as expected.

The problem is simple, Oracle isn't fond of `IFILE`ing files from networked drives. So, to get around this, I needed to map a network drive instead, and use the drive specifier in my `IFILE`.

First map a persistent network drive to be my (new) `Y:` drive. This _should_ be reconnected at logon until further notice. Note that this mapping uses my current credentials to make the connection.

```cmd
net use Y: \\servername\share_name /PERSISTENT:YES
```

And in my `tnsnames.ora`, I now have this:

```
IFILE="Y:\central_tnsnames\tnsnames.ora"
```

And now, it all _just_ works!

```
C:\Users\ndunbar\Downloads>tnsping barney

TNS Ping Utility for 64-bit Windows: Version 11.2.0.1.0 - Production on 19-MAY-2
016 12:21:23

...

Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION= (ADDRESS= (PROTOCOL=TCP) (HOST=bedrock) (PORT=1521)) (CONNECT_DATA= (SERVER=dedicated) (SERVICE_NAME=barney)))
OK (160 msec)
```

HTH
