---
title: "Interesting Data Guard Problem"
date: "2014-01-15"
categories: 
  - "oracle"
---

While checking out a dataguarded database prior to being handed over into production, I needed to test that both `OEM` and `dgmgrl` could carry out a switchover and failover from the (stand-alone) primary db (ORCL_PDB) to the physical standby database (ORCL_SBY), and back again.

### The Problem

> Database and server names have been changed, to protect the innocent, and me!

`OEM` had no problems, other than the usual "bug" whereby the credentials used for the _standby_ server were those for the _primary_ server, but hey, that's `OEM` for you, it's nothing if not inconsistent! However, when I tried to use `dgmgrl` I found a small problem.

While I could happily switchover to the standby database, from either server, switching back always failed with the following error:

```sql
DGMGRL> **switchover to "ORCL_PDB"**

Performing switchover NOW, please wait...
New primary database "ORCL_PDB" is opening...
Operation requires shutdown of instance "ORCL_SBY" on database "ORCL_SBY"
Shutting down instance "ORCL_SBY"...
ORACLE instance shut down.
Operation requires startup of instance "ORCL_SBY" on database "ORCL_SBY"
Starting instance "ORCL_SBY"...
**Unable to connect to database
ORA-12521: TNS:listener does not currently know of instance requested in connect descriptor

Failed.
Warning: You are no longer connected to ORACLE.

Please complete the following steps to finish switchover:
        start up and mount instance "ORCL_SBY" of database "ORCL_SBY"**
```

A quick `srvctl start database -d $ORACLE_SID -o mount` sorted things out while I investigated the problem.

Data Guard requires that there be an entry in tnsnames.ora for both databases and also for a service name consisting of the database and "DGMGRL". I checked.

Both TNSNAMES.ORA files have the following, and all entries are configured correctly:

- ORCL_PDB
- ORCL_PDB_DGMGRL
- ORCL_SBY
- ORCL_SBY_DGMGRL

I had no problems running `sqlplus sys/password@orcl_whatever as sysdba` for any of the above.

Looking in the listener logfile for the standby server's listener, I noticed that there were entries where the error code shown above (ORA-12521 and also ORA-12514)) were present, however, there was a problem with the host_name.

```text
msg time='yyyy-mm-ddThh:mm:ss.fff+00:00' org_id='oracle' comp_id='tnslsnr'
 type='UNKNOWN' level='16' host_id='sby_server'
 host_addr='x.x.x.x'
 15-JAN-2014 12:44:13 * (CONNECT_DATA=(SERVICE_NAME=ORCL_SBY_DGMGRL)(INSTANCE_NAME=**ORCL_XXX**)(SERVER=DEDICATED)(CID=(PROGRAM=dgmgrl)(HOST=sby_server)(USER=oracle))) * (ADDRESS=(PROTOCOL=tcp)(HOST=x.x.x.x)(PORT=53336)) * establish * ORCL_SBY_DGMGRL * 12521
```

The logfile was showing the instance_name - ORCL_XXX - as something completely unrelated to the instance_name for the standby database - ORCL_SBY. Most confusing, especially when I had already confirmed that tnsnames.ora was correct and also that all the entries functioned correctly, from both servers. Where was this erroneous host name coming from?

Looking in `dgmgrl` again, I checked the `StaticConnectIdentifier` property for both databases.

```sql
DGMGRL> show database 'ORCL_PDB' StaticConnectIdentifier

StaticConnectIdentifier =
'(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=pdb_server)(PORT=1522))(CONNECT_DATA=(SERVICE_NAME=ORCL_PDB_DGMGRL)(INSTANCE_NAME=ORCL_PDB)(SERVER=DEDICATED)))'

DGMGRL> show database 'ORCL_SBY' StaticConnectIdentifier

StaticConnectIdentifier =
'(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=sby_server)(PORT=1522))(CONNECT_DATA=(SERVICE_NAME=ORCL_SBY_DGMGRL)(INSTANCE_NAME=ORCL_XXX)(SERVER=DEDICATED)))'
```

Bingo! At least, after starting at the screen for a few minutes, it was bingo! I finally spotted that the stand by database's property had the wrong instance name.

### The Solution

A simple property edit for the standby database was carried out in `dgmgrl` as follows:

```sql
 DGMGRL> edit database 'ORCL_SBY' set property
StaticConnectIdentifier='(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=sby_server)(PORT=1522))(CONNECT_DATA=(SERVICE_NAME=ORCL_SBY_DGMGRL)(INSTANCE_NAME=ORCL_SBY)(SERVER=DEDICATED)))'; 
```

The `edit database` command above is all on one line by the way.

And that was it. After making the change, I was able to run switchovers to and from the standby on eiither server. Job done.
