---
title: "Oracle Text aka CONTEXT - Installing on 11g"
date: "2012-05-14"
categories: 
  - "oracle"
---

It’s supposed to be installed by default, according to the documentation, but for some reason or another, I managed to build a brand new 11.2 database, on Linux, with no `CTXSYS` user present.

### Installing Context

Here’s how to install Oracle Text and the English language defaults, into an 11.2 database.

```sql
SQL> connect / as SYSDBA
SQL> spool ctxsys_installation.log

-- Parameters are password, default t/s, temp t/s, don't lock the account. 
SQL> @?/ctx/admin/catctx.sql secret SYSAUX TEMP NOLOCK

SQL> connect CTXSYS/secret
SQL> @?/ctx/admin/defaults/dr0defin.sql "ENGLISH";

SQL> connect / as SYSDBA
SQL> alter user ctxsys account lock password expire;

SQL> spool off
```

### Removing Context

Here’s how to remove Oracle Text from an 11.2 database.

```sql
SQL> connect / as SYSDBA
SQL> spool ctxsys_removal.log

SQL> @?/ctx/admin/catnoctx.sql

SQL> spool off
```
