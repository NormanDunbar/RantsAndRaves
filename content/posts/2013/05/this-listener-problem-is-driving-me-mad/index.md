---
title: "This Listener Problem is Driving Me Mad!"
date: "2013-05-10"
categories: 
  - "oracle"
  - "listener"
---

I have been looking at this far too long, and I'm stumped. I resolved a similar problem yesterday on another server. That was down to the ORACLE_HOME setting in listener.ora having a '1' in it rather than a '2'. Took ages to spot that.

Anyway, here the stuff you'll need to know to sort this for me, or suggest stuff. It's a question on Oracle L seeing as there is a lot of evidence to post.

As ever, server names etc have been changed to protect the innocent!

**Update** We have a solution! Scroll to the bottom for details.

## Oracle and OS Versions

Oracle Database: Standard Edition, 11.2.0.3 64 bit.
Server: SLES 10 sp 4
Uname -r: 2.6.16.60-0.97.1-smp
hostname: orcl11gserver 

## The Problem

In a word, setting ORACLE_SID and connecting to a user/password works fine. Connecting to user/password@alias gives the following error:

```text
ERROR:
ORA-01034: ORACLE not available
ORA-27101: shared memory realm does not exist
Linux-x86_64 Error: 2: No such file or directory
Process ID: 0
Session ID: 0 Serial number: 0
```

## Database Info

I can connect to the database, both as sysdba and as a non-sysdba user provided I don't use the listener:

```bash
$ sqlplus / as sysdba
...
Connected.
```

```sql
SQL> show parameter listener

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
listener_networks                    string
local_listener                       string
remote_listener                      string

SQL> show parameter db_name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_name                              string      orcl11g

SQL> show parameter service

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
service_names                        string      orcl11g.world

SQL> select * from global_name;

GLOBAL_NAME
--------------------------------------------------------------------------------
orcl11g.WORLD
```

## Listener.ora

```text
lsnr_orcl11g =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS_LIST =
        (ADDRESS = (PROTOCOL = TCP)(HOST = orcl11gserver)(PORT = 1521))
      )
    )
  )

SID_LIST_lsnr_orcl11g =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl11g)
      (ORACLE_HOME = /opt/oracle/product/11.2.0.3/db_1)
      (SID_NAME = orcl11g)
    )
)

DYNAMIC_REGISTRATION_lsnr_orcl11g = off
SUBSCRIBE_FOR_NODE_DOWN_EVENT_lsnr_orcl11g=OFF
```

## Tnsnames.ora

```text
orcl11g,orcl11g.world =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(Host = orcl11gserver)(Port = 1521))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl11g)
    )
  )
```

## Sqlnet.ora

```text
NAMES.DIRECTORY_PATH= (LDAP, TNSNAMES, EZCONNECT, HOSTNAME)
```

## Oratab

```text
orcl11g:/opt/oracle/product/11.2.0.3/db_1/:N
```

## Tnsping

```bash
$ tnsping orcl11g:
```
```text
Used parameter files:
/opt/oracle/product/11.2.0.3/db_1/network/admin/sqlnet.ora

Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS_LIST = (ADDRESS = (PROTOCOL = TCP)(Host = orcl11gserver)(Port = 1521))) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = orcl11g)))
OK (0 msec)
```

## Listener Status

```bash
$ lsnrctl status lsnr_orcl11g
```
```text
...
TNSLSNR for Linux: Version 11.2.0.3.0 - Production
System parameter file is /opt/oracle/product/11.2.0.3/db_1/network/admin/listener.ora
Log messages written to /opt/oracle/diag/tnslsnr/orcl11gserver/lsnr_orcl11g/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=orcl11gserver.testds.ntnl)(PORT=1521)))

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=orcl11gserver)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     lsnr_orcl11g
Version                   TNSLSNR for Linux: Version 11.2.0.3.0 - Production
Start Date                10-MAY-2013 16:51:52
Uptime                    0 days 0 hr. 0 min. 0 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /opt/oracle/product/11.2.0.3/db_1/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/orcl11gserver/lsnr_orcl11g/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=orcl11gserver.testds.ntnl)(PORT=1521)))
Services Summary...
Service "orcl11g" has 1 instance(s).
  Instance "orcl11g", status UNKNOWN, has 1 handler(s) for this service...
The command completed successfully
```

## Where is Listener Running From?

```bash
ps -ef|grep -i ls[n]r_cds
```
```text
oracle   21180     1  0 16:51 ?        00:00:00 /opt/oracle/product/11.2.0.3/db_1/bin/tnslsnr lsnr_orcl11g -inherit
```

## Listener Log

The listener log shows the connection attempt being made, and established ok with a result code of zero.

```text
<msg time='2013-05-10T17:25:56.866+01:00' org_id='oracle' comp_id='tnslsnr'
 type='UNKNOWN' level='16' host_id='orcl11gserver'
 host_addr='10.57.18.116'>
 <txt>10-MAY-2013 17:25:56 * (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=orcl11g)(CID=(PROGRAM=sqlplus)(HOST=orcl11gserver)(USER=oracle))) * (ADDRESS=(PROTOCOL=tcp)(HOST=10.57.18.116)(PORT=12633)) * establish * orcl11g * 0
 </txt>
</msg>
```

## Client Trace

Don't worry, I'm not about to paste an entire ADMIN level trace here. But looking in one, I saw this extract:

```text
nsbasic_brc:type=12, plen=11
nsbasic_brc:what=17, tot =11
nsbasic_brc:packet dump
nsbasic_brc:00 0B 00 00 0C 00 00 00  |........|
nsbasic_brc:01 00 01                 |...     |
nsbasic_brc:exit: oln=0, dln=1, tot=11, rc=0
nioqrc: found a break marker...
nioqrc: Recieve: returning error: 3111
```

This is sort of interesting, as it seems to indicate I got a break from somewhere or something! I saw this on my other similar problem as well, so it's the same in the two trace files, but I solved the other problem by correcting the Oracle Home in `listener.ora`. Not this time!

## The Solution

There are many people on oracle-l who took the time to look at the problem, so thanks to all. There are, however, two people to whom I am extremely grateful. They took mere minutes to discover what had been staring me in the face all day, and the winners are:

- @martinberx on Twitter.
- David Barbour on oracle-l.

Both noticed that in `/etc/oratab`, the Oracle Home path had a trailing slash, while in the `listener.ora`, it did not. Sheesh!

Thanks to both.

## The Fix

The fix was relatively simple:

- With the current (wrong) `oratab` settings in force, shut down the database and the listeners. (The problem affected a number of databases/listeners on this server, not just the one I used in the above example.)
- Edit `oratab` to remove the trailing slash.
- Restart the listeners and databases with the new improved `oratab`.
- Test - it all "just works".

:-)
