---
title: "How to Screw Up DB Upgrade Assistant"
date: "2012-06-21"
categories: 
  - "oracle"
---

It was my own fault, but in case it proves even slightly useful....

I was upgrading from 11202 to 11203 Enterprise using the _DB Upgrade Assistant_ utility. When I said 'go do it' it went off, chugged for a bit, then barfed. DBUA informed me that the database wasn't running. I checked, it was.

Cutting a long story short, I checked the indicated logfile and discovered that DBUA had connected to the database but then got a couple of errors telling it that 'oracle was not available'. Hmm.

Turns out that I'd added a couple of SQL statements to `glogin.sql`, with

```sql
alter session set nls_date_format = 'dd/mm/yyyy hh24:mi:ss';
```

being one of the offending statements. This works perfectly as long as you connect while the database(s) are open but, if a database is shut, and you login as sysdba to start it, `glogin.sql` is still executed, and because the database is down, the SQL statement(s) fail.

DBUA picked up the failure and refused to carry on.

Removing the SQL statement(s) from `glogin.sql` fixed the problem.

The moral to this tale is simple, don't put SQL in `glogin.sql` (or `login.sql`) if you ever shut down your databases!

In case you are wondering, `glogin.sql` lives in `$ORACLE_HOME/sqlplus/admin` and will be executed on each successful connection to a database with SQL*Plus. `Login.sql` will also be executed on every successful connection to a database (at least, from 10g onwards) _after_ `glogin.sql`, but only if it is found on `$SQLPATH`.
