---
title: "Wondering Why The Oracle Databases Won't Start With A SLES Reboot?"
date: "2011-06-24"
categories: 
  - "linux"
  - "oracle"
---

Me too. Took ages to hit the "duh" moment, then it became pretty obvious! The file `/etc/init.d/oracle` also known as `rcoracle` to root users can be used to do a number of things such as starting the databases, starting the (default LISTENER) listener, CRS etc but, as I eventually found out, you have to configure it to do so!

The configuration file is `/etc/sysconfig/oracle`.

Most of the options are defaulted to off, except for the setting of kernel parameters (`SET_ORACLE_KERNEL_PARAMETERS="yes"`) which is very useful.

Cheers.
