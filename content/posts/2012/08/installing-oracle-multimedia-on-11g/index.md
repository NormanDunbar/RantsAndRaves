---
title: "Installing Oracle Multimedia on 11g"
date: "2012-08-17"
categories: 
  - "oracle"
---

Installing Oracle Multimedia, which is required for [Spatial and/or Locator](/posts/2012/05/installing-locator-or-spatial-on-11g/ "Installing Locator or Spatial on 11g") is quite simple.

All of the following must be carried out while logged in as a SYSDBA user.

```sql
SQL> spool ordinst.log
SQL> @?/ord/admin/ordinst SYSAUX SYSAUX
...
...
SQL> spool off

SQL> spool catim.log
SQL> @?/ord/im/admin/catim
...
...
SQL> spool off
```

Lots of _stuff_ will scroll up the screen but will also be copied to the spool files named. Check those for obvious errors, then check to see if it worked as follows:

```sql
SQL> select comp_id, version, status
  2  from dba_registry
  3  where comp_id = 'ORDIM';

COMP_ID    VERSION        STATUS
---------- ---------- ---------
ORDIM      11.2.0.3.0     VALID
```

You can also force check the validity as follows, should you ever need to:

```sql
SQL> set serveroutput on

SQL> execute sys.validate_ordim;
PL/SQL procedure successfully completed.
```

If there are any invalid objects in the ORDIM component, the above procedure will display a message about them and set the status column in the `DBA_REGISTRY` to INVALID. You should check the status after running the above.
