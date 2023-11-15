---
title: "Using Oracle 11g Adrci for Incident Reporting"
date: "2012-08-22"
categories: 
  - "oracle"
---

`Adrci` is a new tool in Oracle 11g which makes life a little easier when gathering evidence to send off to Oracle Support, but it can make life easier when you simply wish to view the alert log, for example.

As ever, you need to be logged in to the database server and have the environment set in the normal `oraenv` manner. This is how we used to do it in the old, pre 11g, days:

```sql
SQL> connect / as sysdba
connected

SQL> show parameter background_dump_dest

NAME                  TYPE    VALUE
-------------------- ------ -----------------------------------------
background_dump_dest  string  /srv/xxx/oradata/yyyyy/diag/rdb
                              ms/yyyyy/yyyyy/trace
SQL> exit
```

```bash
$ view /srv/xxx/oradata/yyyyy/diag/rdbms/yyyyy/yyyyy/trace/alert_yyyyy.log
```

In modern times, with 11g of course, we no longer have to do this, as we have `adrci` instead, as follows:

```bash
$ adrci

ADRCI: Release 11.2.0.3.0 - Production .....
...
ADR base = "/srv/xxx/oradata/yyyyy"

adrci> show alert
```

That's all there is to it. The alert log is, on my server, displayed in the `vi` editor and when you quit with the `:q` command, you return to `adrci`.

While you are there, you can check for any incidents recorded in the alert log:

```bash
adrci> show incident

ADR Home = /srv/xxx/oradata/yyyyy/diag/rdbms/yyyyy/yyyyy
**********************************************************************
INCIDENT_ID  PROBLEM_KEY       CREATE_TIME
------------ ----------------- ---------------------------------
145          ORA 227           2012-08-06 16:22:40.026000 +01:00
40165        ORA 600           2012-08-10 13:14:23.033000 +01:00
```

Given that ORA-600, I can get more detail about it with the following command:

```bash
show incident -mode detail -p "incident_id=40165"

ADR Home = /srv/xxx/oradata/yyyyy/diag/rdbms/yyyyy/yyyyy
**********************************************************************

**************************************
INCIDENT INFO RECORD 1
**************************************
  INCIDENT_ID       40165
  STATUS            ready
  PROBLEM_ID        4
...
```

The `-mode` option can be one of:

- Basic - simply lists the appropriate line(s) that you would see in a `show incident` command output. Not much use to be honest.
- Brief - gives a bit more detail that basic. Doesn't include details of any trace files created for this incident.
- Detail - gives lots of information including details of all trace files created for this incident.

The `-p` option is simply a `where` clause on any of the columns in the incident table. You can obtain a list of columns as follows:

```bash
adrci> describe incident

Name          Type          NULL?
------------- ------------- ------
INCIDENT_ID   number
PROBLEM_ID    number
CREATE_TIME   timestamp
CLOSE_TIME    timestamp
STATUS        number
FLAGS         number
....
```

The `-p` options can be ANDed and/or ORed together:

```bash
adrci> show incident -p "incident_id = 40165 or incident_id = 145"
```

Given that incident 40165 is an ORA-600, it should be sent off to Oracle Support for diagnosis, so I need to gather all the evidence together. Remember how that was done in the old days? Well, this is how simple it is with `adrci`:

First, create a new package:

```bash
adrci> ips create package
Create package 7 without any contents, correlation level typical
```

That creates a blank package, with nothing in it. You now add all the incidents you need to add:

```bash
adrci> ips add incident 40165 package 7
Added incident 40165 to package 7
```

That has added the incident to the package's metadata only, there are still no actual package files created. You may add further incidents to the package as desired.

When done adding incidents, generate a physical package for support:

```bash
adrci> ips generate package 7
Generate package 7 in /home/oracle/IPSPKG_20120820104439_COM_7.zip, mode complete
```

The zip file name created contains everything Oracle Support will need, all the trace files, the alert log, various metadata files and so on. This file can be uploaded to Oracle Support.

The file is created in the current working directory. If you want to specify where to put it, the following command will help:

```bash
adrci> ips generate package 7 in '/srv/xxx/oradata/yyyyy'
Generate package 7 in /srv/xxx/oradata/yyyyy/IPSPKG_20120820104842_COM_7.zip, mode complete
```

That concludes this _very_ brief overview of the `adrci` utility. For more details check out the _Database Administrator's Guide_, Chapter 9 - _Managing Diagnostic Data_ Have fun.
