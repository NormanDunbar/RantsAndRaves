---
title: "Where Does OraInventory Live? Can I Move it?"
date: "2012-11-30"
categories: 
  - "linux"
  - "oracle"
---

Looking for the location of oraInventory on a server? Want to know where it is? Read on.

There is a file, known to Oracle, which holds the location of the inventory. Of course, it isn't in the same place on every server, but the ones I know of have it as follows:

- Linux: `/etc/oraInst.loc`
- HP-UX: `/var/opt/oracle/oraInst.loc`
- Windows: Registry at `HKLM/software/oracle/inst_loc`

For any other Unix, you can find it (as the oracle or root user) with:

```bash
find / -type f -name oraInst.loc -print
```

Once you know where it lives, you can move it simply. The following example moves it from the current location, found in `oraInst.loc` to `/opt/oracle/oraInventory`:

```bash
$ cat /etc/oraInst.loc

inventory\_loc=/u01/app/oracle/oraInventory
inst\_group=oinstall

$ cd /opt/oracle
$ cp -Rp /u01/app/oracle/oraInventory ./

$ vi /etc/oraInst.loc
  :1
  s?/u01/app?/opt?
  :wq

$ cat /etc/oraInst.loc

inventory\_loc=/opt/oracle/oraInventory
inst\_group=oinstall
```

Job done! Although it might be wise to take a backup of the original location, just in case, and then delete it from the old location:

```bash
$ cd /opt/oracle
$ tar -cvzf u01.app.oracle.tgz /u01/app/oracle
...

$tar -tzf u01.app.oracle.tgz   ## Just checking ...
...

$cd /u01/app/
$pwd                           ## Safety check!
/u01/app

$ls                            ## Another safety check!
oracle

$rm -Rf oracle                 ## Getting nervous yet? I am!
```

And that it, all done.
