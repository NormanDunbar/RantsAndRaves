---
title: "Oracle RAC - Flashback Database"
date: "2014-09-10"
categories: 
  - "oracle"
---

It was a simple enough request, flashback this particular database to a guaranteed restore point. What could possibly go wrong?

Database names etc have been changed to protect the innocent, and me!

```sql
$ sqlplus / as sysdba

SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

SQL> startup mount
ORACLE instance started.
...
Database mounted.

SQL> flashback database
  2  to restore point GRP_2014_05_17_13_10;
```

However, this raised the following error:

```text
ORA-38748: cannot flashback data file 1 - file is in use or recovery
ORA-01110: data file 1: '+DATA/XXXXXX/datafile/system.269.759338709'
```

A _minor_ panic then ensued! It's a production database, and I have limited downtime allocated!

In the back of my brain, I thought "maybe it's a RAC database"?

```sql
SQL> show parameter cluster

NAME                       TYPE        VALUE
-------------------------------------- -----
cluster_database           boolean     TRUE
cluster_database_instances integer     2

SQL> show parameter instance_name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
instance_name                        string      XXXXXX_1
```

Ok, It _is_ RAC, there are two instances, and I'm on instance XXXXXX_1. It is _obviously_ still in use by the other instance which will be XXXXXX_2 using the naming conventions for this system. I need to bring it all down. The easiest way is to use `srvctl`:

```sql
SQL> exit
```

```bash
$ srvctl stop database -d XXXXXX -o immediate
$sqlplus / as sysdba
```

```sql
SQL> startup mount
ORACLE instance started.
...
Database mounted.

SQL> flashback database to restore point GRP_2014_05_17_13_10;
Flashback complete.

SQL> alter database open resetlogs;
Database altered.
```

All I need to do now is start the other instance:

```bash
$ srvctl start instance -d XXXXXX -i XXXXXX_2
```

Job done, and still within the downtime!

And here's a quick tip. Because the database is effectively right back at 17th May 2014, the guaranteed restore point and its numerous archived and flashback logs, which are taking up space in the FRA, can be dropped and recreated to save FRA space:

```sql
SQL> drop restore point GRP_2014_05_17_13_10;
Restore point dropped.

SQL> create restore point GRP_2014_05_17_13_10 guarantee flashback database;
Restore point created.
```

