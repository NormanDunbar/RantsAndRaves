---
title: "Using \"chopt\" to Enable and Disable Oracle Options"
date: "2012-11-23"
categories: 
  - "oracle"
---

As you may know, Oracle databases come with a number of options. Some of these cost extra and if inadvertantly installed, Oracle must be paid money - note, you don't have to be using them, only have them installed, to require payment. So what do you do if you need to remove an option?

In the old days, you used to have to rebuild the oracle binaries to add or remove options. To do this you needed to know the names of a number of _make targets_ - not for the faint hearted.

From 11g onwards, the process is much simpler. Oracle now supply the `chopt` (change option) utility to make your DBS's life simple.

The utility is supplied with Enterprise and Standard Editions. It allows you to enable or disable the following options:

- Data Mining
- Database Vault
- Oracle Label Security
- OLAP
- Partitioning
- Real Application Testing

You can run the utility with no parameters to see what it does and how you should call it in anger:

```bash
$ chopt

usage:

chopt <enable|disable> <option>

Options:

                  dm = Oracle Data Mining RDBMS Files
                  dv = Oracle Database Vault option
                lbac = Oracle Label Security
                olap = Oracle OLAP
        partitioning = Oracle Partitioning
                 rat = Oracle Real Application Testing

e.g. chopt enable rat
```

So, for example, to disable partitioning because the junior DBA has mistakenly installed it (all these options are selected and installed by default in Enterprise Edition!) then all you do is set the correct Oracle Home using `oraenv` in the normal manner, then:

`chopt disable partitioning` as the following example demonstrates:

```bash
$ chopt disable partitioning

Writing to /srv/oracle/product/11gR1/db/install/disable\_partitioning.log...
/usr/bin/make -f /srv/oracle/product/11gR1/db/rdbms/lib/ins\_rdbms.mk part\_off ORACLE\_HOME=/srv/oracle/product/11gR1/db
/usr/bin/make -f /srv/oracle/product/11gR1/db/rdbms/lib/ins\_rdbms.mk ioracle ORACLE\_HOME=/srv/oracle/product/11gR1/db
```

You can check the log file named on the first line of output for details. Enabling an option is just as simple:

```bash
$ chopt enable partitioning

Writing to /srv/oracle/product/11gR1/db/install/enable\_partitioning.log...
/usr/bin/make -f /srv/oracle/product/11gR1/db/rdbms/lib/ins\_rdbms.mk part\_on ORACLE\_HOME=/srv/oracle/product/11gR1/db
/usr/bin/make -f /srv/oracle/product/11gR1/db/rdbms/lib/ins\_rdbms.mk ioracle ORACLE\_HOME=/srv/oracle/product/11gR1/db
```

You can only enable or disable a single option at a time, unlike when you are running the old style `make` commands where you could specify to turn them all on or off in one go. Progress?
