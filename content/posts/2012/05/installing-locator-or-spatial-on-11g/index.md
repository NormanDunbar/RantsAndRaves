---
title: "Installing Locator or Spatial on 11g"
date: "2012-05-15"
categories: 
  - "oracle"
---

Locator is "Spatial Lite" if you wish, and costs nothing. It can be installed in Standard or Enterprise Editions with no additional licensing costs. You cannot do everything in Locator that you can in Spatial - but what do you expect for free? ;-)

To install **Locator** you need to be aware that things changed at 11g, so what _used_ to work on 10g no longer does on 11g and can lead to you having a huge mess of objects and types to hunt down and remove from your SYS account - ask me how I know!

Note: If you have already [installed Oracle Multimedia](http://qdosmsq.dunbar-it.co.uk/blog/2012/08/installing-oracle-multimedia-on-11g/ "Installing Oracle Multimedia on 11g"), the `MDSYS` user will already exists. In this case, simply skip the create user command in the following.

```sql
SQL> connect / as sysdba

SQL> create user mdsys identified by secret
  2  default tablespace sysaux;

SQL> @?/md/admin/mdprivs.sql

SQL> connect mdsys/secret
SQL> @?/md/admin/catmdloc

SQL> connect / as sysdba
SQL> alter user mdsys
  2  account lock
  3  password expire;
```

You can read the details about Locator in the file `$ORACLE_HOME/md/doc/README_LOCATOR.txt` - but don't expect too much assistance, it doesn't even mention how to install it!

You won't be able to find Locator in `DBA_REGISTRY` nor will you be able to find it in `V$OPTION` either.

```sql
SQL> col parameter format a20
SQL> col value format a5

SQL> select parameter, value
  2  from v$option
  3  where parameter like 'Spatial%'
  4  or parameter like 'Locator%';

PARAMETER            VALUE
-------------------- -----
Spatial              FALSE
```

You can see that Locator doesn't turn up in the output - that's because it's not an option, unlike Spatial.

My Oracle Support (MOS) has a document that explains how to detect if Locator is installed. The document id is 357943.1. Unfortunately, the process explained doesn't work. Sigh.

You _cannot_ even assume Locator is installed if Spatial is not installed _and_ you have located a schema named `MDSYS` which owns objects, because [installing Oracle Multimedia](/posts/2012/08/installing-oracle-multimedia-on-11g/ "Installing Oracle Multimedia on 11g") creates the `MDSYS` user if it is not there already.

* * *

Installing **Spatial** requires that you are running Enterprise Edition and that you didn't deselect it when installing the software originally. If you did, you need to re-run the installer and when you see the `Enterprise Options` button, click it and select the Spatial component.

If you don't do this then the file `catmd.sql` and it's associated helpers, will not be copied to `$ORACLE_HOME/md/admin`.

Spatial costs extra dosh and must be licensed separately from your Enterprise Edition software.

```sql
SQL> connect / as sysdba

SQL> create user mdsys identified by secret
  2  default tablespace sysaux
  3  account lock
  4  password expire;

SQL> @?/md/admin/mdprivs.sql

SQL> @?/md/admin/catmd
```

You can read the details about Spatial in the file `$ORACLE_HOME/md/doc/README.txt`.

As with Locator, you won't find Spatial in `DBA_REGISTRY` so, as before, you must look in `V$OPTION`.

```sql
SQL> col parameter format a20
SQL> col value format a5

SQL> select parameter, value
  2  from v$option
  3  where parameter like 'Spatial%';

PARAMETER            VALUE
-------------------- -----
Spatial              TRUE
```
