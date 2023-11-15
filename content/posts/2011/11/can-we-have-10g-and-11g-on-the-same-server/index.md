---
title: "Can We Have 10g and 11g on the Same Server?"
date: "2011-11-02"
categories: 
  - "oracle"
  - "rants-raves"
---

If you install 10g and 11g on the same server, which one do you wish to supply the executables for "oraenv"?

If you install 10g first and allow 11g to overwrite the files in /usr/local/bin when you run "root.sh" then when you eventually call oraenv to set a 10g environment, you get a warning that "$ORACLE\_HOME/bin/orabase" cannot be found.

If you allow the 10g files to overwrite the 11g ones, you don't set ORACLE\_BASE. I wonder what problems that might cause?

Cheers.
