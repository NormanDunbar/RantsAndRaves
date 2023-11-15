---
title: "Oracle Change The Patching Process From 11.2.0.2"
date: "2010-09-13"
categories: 
  - "oracle"
---

Oracle have made subtle changes to the way that they advise you to patch an installation from the release of 11.2.0.2. The full gory details can be found [here](https://supporthtml.oracle.com/ep/faces/secure/km/DocumentDisplay.jspx?id=1189783.1 "MOS Document 1189783.1") (My Oracle Support link - support contract required) but in brief:

- You are now advised to perform an out-of-line patch, however, you can do an in-line patch if you are short of space. However, this will delete the existing installation. Beware & take copious backups!
- The patch kit is a full installation - so you get  the latest patch kit and install it instead of getting  the base release and then patching it.
- Grid components _must_ be patched out-of-line.

Cheers.
