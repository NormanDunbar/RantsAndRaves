---
title: "Removing \"Rogue\" Expensive Oracle Options After Upgrading to 10.2.0.x."
date: "2011-11-02"
categories: 
  - "oracle"
  - "rants-raves"
---

Install 10.2.0.1 - the base release - of Oracle and deselect the various Enterprise Edition Options - these cost money and we don't like that, especially if we don't use them.

Patching to 10.2.0.5 (in my installation - it could be different with previous versions) silently adds OLAP, Data Mining etc back into the mix. Not good - especially if/when Oracle decide to audit your licenses. You pay for what you _install_ whether used or not.

So, to remedy the problem, do this:

```bash
cd $ORACLE_HOME/rdbms/lib 
make -f ins_rdbms.mk olap_off rat_off rac_off dm_off sdo_off ioracle
```

You may see an error telling you that SDO can't be turned off. That's fine, at least we tried! The various options are:

- olap_off = OLAP
- rat_off = Real Application Testing
- rac_off = Real Application Cluster
- dm_off = Data Mining
- sdo_off = Spatial

There are others in the make file, but these are the ones I'm mostly interested in not having!

Cheers.
