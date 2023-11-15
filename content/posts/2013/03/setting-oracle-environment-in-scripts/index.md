---
title: "Setting Oracle Environment in Scripts"
date: "2013-03-18"
categories: 
  - "linux"
  - "oracle"
---

A quickie! How do you set the correct Oracle environment in scripts? Do you hard code? You'd better not.

I've lost count of the times I've ended up with, for example, a 10g database running with bits of the 9i software hanging around. It leads to monumental problems that can be hard to track down.

Moral: _Do not hard code Oracle environment details._

This is what I usually do:

```bash
#!/usr/bin/env bash

export ORAENV_ASK=NO
export ORACLE_SID=my_sid
. oraenv
export ORAENV_ASK=YES

# Rest of bash script goes here....
```

Reagrdless of how many times the database version gets an update, you will still always have the correct Oracle software on the path and in the environment when the script runs.

Today, I learned a new way to do the above, with much less typing. You can see an example [here on Gokhan Atil's Data Blog](http://www.gokhanatil.com/2013/03/bash-script-to-upload-rman-backups-via-ftp.html "http://www.gokhanatil.com/2013/03/bash-script-to-upload-rman-backups-via-ftp.html"). (**Update 23/02/2023**: The linked page is no more, it has ceased to be!)

```bash
#!/usr/bin/env bash

. oraenv <<EOF
my_sid
EOF

# Rest of bash script goes here....
```

All those years of Oracle and I never figured out that I could user a "here" document. Thanks Gokhan.
