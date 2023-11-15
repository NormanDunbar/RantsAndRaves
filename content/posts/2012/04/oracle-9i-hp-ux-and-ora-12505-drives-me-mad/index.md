---
title: "Oracle 9i, HP-UX and ORA-12505 Drives Me Mad!"
date: "2012-04-23"
categories: 
  - "oracle"
---

I've just created a new database on an HP-UX server. The database is Oracle 9i (yes, I know, I know!) and no matter what I do, I can't connect via the listener without getting the dreaded _ORA-12505: TNS:listener could not resolve SID given in connect descriptor_ error message.

I've done this lots of times in the past, but for some reason, I can't get it to work today. It's driving me mad!

The reason I have to use a `local_listener` parameter on HP-UX is because the `hostname` and the `uname -n` commands don't return the same result as the hostname is longer than 8 characters:

```sql
SQL> !hostname
myserver0011

SQL> !uname -n
myserver
```

So, we have to use a `local_listener` setting in the spfile, and register the database with that listener because we can't connect with `user/password@alias` otherwise.

```sql
SQL> connect my_user/secret@my9idb
ERROR:
ORA-12505: TNS:listener could not resolve SID given in connect descriptor
Warning: You are no longer connected to ORACLE.
```

Hmmm, it works fine without using the listener:

```sql
SQL> conn / as sysdba
Connected.
```

Some sanity checks:

```sql
SQL> show parameter db_name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_name                              string      my9idb

SQL> show parameter local_listener

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
local_listener                       string      (ADDRESS=(PROTOCOL=TCP)(HOST=m
                                                 yserver0011)(PORT=1556))

SQL> !tnsping my9idb
```
```text
TNS Ping Utility for HPUX: Version 9.2.0.6.0 - Production on 23-APR-2012 16:12:30

Copyright (c) 1997 Oracle Corporation.  All rights reserved.

Used parameter files:
/opt/oracle/product/9.2.0.6/db/network/admin/sqlnet.ora

Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=myserver0011)(PORT=1556)) (CONNECT_DATA=(SID=my9idb)(SERVER=DEDICATED)))
OK (10 msec)
```

```sql
SQL> !ping myserver0011
PING myserver0011: 64 byte packets
64 bytes from 10.55.127.122: icmp_seq=0. time=0. ms
64 bytes from 10.55.127.122: icmp_seq=1. time=0. ms
64 bytes from 10.55.127.122: icmp_seq=2. time=0. ms

----myserver0011 PING Statistics----
3 packets transmitted, 3 packets received, 0% packet loss 
round-trip (ms)  min/avg/max = 0/0/0
```

```sql
SQL> !lsnrctl services lsnr_my9idb
```
```text
LSNRCTL for HPUX: Version 9.2.0.6.0 - Production on 23-APR-2012 16:15:51

Copyright (c) 1991, 2002, Oracle Corporation.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=myserver0011)(PORT=1556)))

The listener supports no services
The command completed successfully
```
The clue was in the second last line, _the listener supports no services_. But I missed it the first time I checked, and the second, third .....

It eventually dawned on me:

```sql
SQL> alter system set service_names='my9idb' scope=both;
System altered.

SQL> alter system register;
System altered.

SQL> conn myuser/secret@my9idb
Connected.
```

Result!

It appears that the `alter system register` command isn't required, as soon as I set the `service_names` parameter, the listener seems to notice. However, I'm taking no chances after today!

_All usernames, passwords, IP addresses and server names in this article have been disguised to protect the innocent!_
