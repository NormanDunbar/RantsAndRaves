---
title: "How To Extract Details From /etc/oratab on Linux"
date: "2011-06-24"
categories: 
  - "linux"
  - "oracle"
---

Ever wanted to parse `/etc/oratab` but ignore all the comments and blank lines? So did I. Here's how ...

I can't claim all the credit for this, it is based on something I was doing plus a bit of "stolen" code from [SLES](https://www.suse.com/products/server/ "SUSE Linux Enteprise Server").

```bash
OLDIFS=$IFS
IFS=:
grep -v '^\(#\|$\)' /etc/oratab        |\
while read ORASID ORAHOME AUTOSTART
do
        ## Do what you like here with
        ##  $ORASID, $ORAHOME and $AUTOSTART ##
done
IFS=$OLDIFS
```

Cheers,  
Norm.
