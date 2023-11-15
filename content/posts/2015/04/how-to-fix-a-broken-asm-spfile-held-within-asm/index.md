---
title: "How to Fix a Broken ASM SPFILE, held within ASM"
date: "2015-04-17"
categories: 
  - "oracle"
---

My server rebooted itself and when it came back up, none of the databases or ASM had restarted. Everything is 11.2.0.3 or 11.2.0.1 with ASM being 11.2.0.3 - so Oracle Restart _should_ have kicked in.

> As usual, any identifying names, servers, domains, databases etc have been obfuscated to protect the innocent.

```sql
$srvctl start asm
PRCR-1079 : Failed to start resource ora.asm
CRS-5017: The resource action "ora.asm start" encountered the following error:
ORA-00119: invalid specification for system parameter LOCAL_LISTENER
ORA-00132: syntax error or unresolved network name 'myserver.mydomain.net:1899'
```

The `LOCAL_LISTENER` parameter is incorrect, it should be 'myserver.mydomain.com:1899' with a '.com' and not '.net'.

We have a problem in the spfile that needs to be fixed. Where is it located so that it can be converted to a pfile and corrected? The usual place to check is `$ORACLE_HOME/dbs`.

```bash
$cd $ORACLE_HOME/dbs
$ls spfile*
spfile* not found
```

It isn't in the normal location, what does Oracle Restart know?

```bash
$srvctl config asm -a | grep -i spfile
Spfile: +DATA/asm/asmparameterfile/registry.123.123456789
```

The spfile name may also be listed in the alert.log as part of a startup. It is for me in this case:

```bash
$grep "^Using.*spfile" alert_+ASM.log | tail -1 
Using parameter settings in server-side spfile +DATA/asm/asmparameterfile/registry.123.123456789
```

Now we have a "Catch 22 chicken and egg" problem. The spfile is located inside ASM and we can't start ASM to extract and fix it, because we need the (broken) parameter file to start ASM.

There are numerous blog postings on the internet that explain how to start ASM, or extract the spfile, when the spfile it needs to start is in ASM, but due to a missing `$GRID_HOME/gpnp/myserver/profiles/peer/profile.xml` file, those were not an option here. (I think the problem is that the `profile.xml` is used by RAC only.)

On a normal database, you can create a pfile from the spfile even if the instance is not running. Will that work?

```sql
$sqlplus / as sysasm
Connected to an idle instance.

SQL> create pfile='/home/oracle/pfile.ora' from spfile='+DATA/asm/asmparameterfile/registry.123.123456789';
create pfile='/home/oracle/pfile.ora' from spfile='+DATA/asm/asmparameterfile/registry.123.123456789'
*
ERROR at line 1:
ORA-01565: error in identifying file '+DATA/asm/asmparameterfile/registry.123.123456789'
ORA-17503: ksfdopn:2 Failed to open file +DATA/asm/asmparameterfile/registry.123.123456789
ORA-01034: ORACLE not available
```

That was expected, but it had to be tried!

### Method 1

Maybe a default pfile can be created from the alert log's listing of the non-default startup parameters from the last time it started?

```bash
$cd /app/oracle/diag/asm/+asm/+ASM/trace
$view alert_+ASM.log

...
Using parameter settings in server-side spfile +DATA/asm/asmparameterfile/registry.123.123456789
System parameters with non-default values:
  large_pool_size          = 12M
  instance_type            = "asm"
  remote_login_passwordfile= "EXCLUSIVE"
  local_listener           = "myserver.mydomain.com:1899"
  asm_diskstring           = "/dev/oracleasm/disks/disk*"
  asm_diskgroups           = "FRA"
  asm_power_limit          = 1
  diagnostic_dest          = "/app/oracle"
USER (ospid: 9251): terminating the instance due to error 119
Instance terminated by USER, pid = 9251
```

So that's one way of extracting the non-default startup parameters into a temporary pfile, for those awkward times when you cannot get at the spfile to start ASM as the spfile is located within ASM itself. Extract the above settings from the alert.log and startup with that temporary pfile. Once started, create a new spfile, update Oracle Restart and Robert is your mother's brother.

However, depending on how long ASM has been up, what's to say that _any_ of the listed parameters are still valid? After all, since startup, someone changed the `LOCAL_LISTENER` parameter and it was only when the instance _next_ started up that the foul up became apparent.

### Method 2

There is another way. Thinking, as they say _outside the box_ (Yuk! I avoid cliches like the plague!) about how tnsnames.ora allows `IFILE` commands, suggests that _perhaps_ Oracle might allow me to create a pfile which specifies the existing spfile name _and_ lets me set the correct `LOCAL_LISTENER` parameter to overwrite the broken setting in the spfile?

I confess, I also had a very vague recollection from way back when spfiles were first introduced, that I had seen/read/heard/tried something like this already, but as mentioned, it was a very vague recollection! Nevertheless, let's create a plain vanilla pfile:

```bash
$vi /home/oracle/initASMtemp.ora

*.spfile="+DATA/asm/asmparameterfile/registry.123.123456789"
*.LOCAL_LISTENER='myserver.mydomain.com:1899'
```

The correction goes _after_ the spfile, so that it takes effect rather than being overridden by the broken one in the spfile - assuming this trick works!

```bash
$sqlplus / as sysasm
Connected to an idle instance.

SQL> startup pfile='/home/oracle/initASMtemp.ora';
ASM instance started

Total System Global Area  283930624 bytes
Fixed Size                  2181896 bytes
Variable Size             256582904 bytes
ASM Cache                  25165824 bytes
ASM diskgroups mounted
```

We have a running ASM system!

Fix the broken parameter in the existing spfile:

```sql
SQL> alter system set local_listener='myserver.mydomain.com:1899' scope=spfile;
System altered.

SQL> show parameter local

NAME             TYPE        VALUE
---------------- ----------- -------------------------------
local_listener   string      myserver.mydomain.com:1899
```

A shutdown and restart later and the spfile is once more working correctly.
