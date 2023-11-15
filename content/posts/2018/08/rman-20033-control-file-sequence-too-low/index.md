---
title: "RMAN-20033: control file SEQUENCE# too low"
date: "2018-08-08"
categories: 
  - "oracle"
---

Have you ever seen the error **RMAN-20033: control file SEQUENCE# too** **low** and wondered what could be causing it? 

If you look on MOS, you will probably see that the error is caused by _the control file in use is older than the one that was most recently used to synchronise the RMAN catalogue_ and that you should either recreate the database control file(s), or, delete the database from the catalog and add it in again.

Think again! The problem _could_ be caused by the fact that there are two backups attempting to run at the same time, and both need to synchronise with the control file.

This is especially true if you find that while the error is reproducible, it is intermittent in nature - it doesn't fail every time, which it would if the control file was _really_ to blame.
