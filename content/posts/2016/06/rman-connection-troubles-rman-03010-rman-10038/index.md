---
title: "RMAN Connection Troubles, RMAN-03010 & RMAN-10038"
date: "2016-06-24"
categories: 
  - "oracle"
---

For no reason, after many weeks of use, RMAN suddenly cannot connect:

```
rman target sys/******@dbadb01 catalog ...

...
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-00601: fatal error in recovery manager
RMAN-03010: fatal error during library cache pre-loading
RMAN-10038: database session for channel default terminated unexpectedly
```

Setting debug and trace on the command line has no effect, there is nothing of use in the trace file.

```
rman target sys/****** debug all trace=trace.log
```

The contents of `trace.log` after this were as follows, which is pretty much normal, except for the error messages at the end. No help at all in other words.

```
DBGMISC:    ENTERED krmksimronly [09:55:45.190] 
... 
Calling krmmpem from krmmmai 
RMAN-00571: =========================================================== 
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS =============== 
RMAN-00571: =========================================================== 
RMAN-00601: fatal error in recovery manager 
RMAN-03010: fatal error during library cache pre-loading 
RMAN-10038: database session for channel default terminated unexpectedly
```
Setting event 10046 on the entire database with `alter system ...` and/or running a session 10046 trace for SYS logins using a database trigger also revealed nothing. Not even a trace file for any RMAN sessions.  

Much searching with Google resolved nothing. My MOS account is not connected yet, despite requests, to the current support identifier, so I can't go hunting for a fix on MOS. And the admins are off today as well.  However, this was a subtle clue:  

```
set oracle_sid=dbadb01 
rman target / catalog ... 
... 
connected to target database: dbadb02 (DBID=1170775433) 
...
```

Really? _Dbadb**02**_ 02? I asked for 01, so what's going on?  

The database, dbadb02, was cloned, using RMAN yesterday. It should have a new DBID and such like, so let's check:  

```
tnsping dbadb01  
Used TNSNAMES adapter to resolve the alias 
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = test_server)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = dbadb01))) OK (20 msec)
```

So far so good, next:  
```
tnsping dbadb02  
Used TNSNAMES adapter to resolve the alias 
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = test_server)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = dbadb02))) OK (20 msec)  
```

That looks fine too. Next, check the database:  
```
set oracle_sid=dbadb02 
sqlplus / as sysdba  
select name,value from v$parameter 
where lower(value) like '%dbadb01%';

NAME	VALUE
------------------------------------------------------ 
instance_name     dbadb01 
service_names     dbadb01 
dispatchers      (PROTOCOL=TCP) (SERVICE=dbadb01XDB) 
audit_file_dest	 C:\ORACLEDATABASE\ADMIN\dbadb01\ADUMP
```

Bingo! RMAN doesn't change these parameters after a clone, so the database needs fixing. It looks like RMAN is attempting to connect to the dbadb02 database with the service name of dbadb01, rather than connecting to dbadb01 on that service name.  

There was supposed to be a script executed by the clone script, to fixup those parameters but obviously it didn't work, or some other fault occurred (The DBA responsible will getting a slapped wrist shortly!) - log files will need to be checked!  

A quick fix later:  

```sql
alter system set instance_name='dbadb02' scope=spfile; 
alter system set service_names='dbadb02' scope=spfile; 
alter system set dispatchers='(PROTOCOL=TCP) (SERVICE=dbadb02XDB)' scope=spfile; 
alter system set audit_file_dest='C:\ORACLEDATABASE\ADMIN\dbadb02\ADUMP' scope=spfile; 
startup force  
...
select name,value from v$parameter 
where lower(value) like '%dbadb01%';  

no rows selected  
```

And now, does RMAN work?  

```
rman target sys/******@dbadb01 
... 
connected to target database: dbadb01 (DBID=673233917)  
```

Result!
