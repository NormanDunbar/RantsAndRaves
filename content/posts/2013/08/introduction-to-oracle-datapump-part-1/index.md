---
title: "Introduction to Oracle Datapump - Part 1"
date: "2013-08-31"
categories: 
  - "oracle"
---

Oracle _Datapump_, aka `expdp` and `impdp` were introduced at Oracle 10g to replace the old faithful `exp` and `imp` utilities. Many DBAs around the world find that it's hard to change from what we know like the back of our hand, to something new. We _need_ to change because `exp` is deprecated from 10g onwards and might even already have vanished from 12c - which I have to install as one of my upcoming tasks. The following is a quick introduction for people like me - running Oracle on Linux and slightly averse to change! ;-)

## Introduction to Datapump

> All of the following is based on 11.2.0.2 Enterprise Edition of Oracle, running on Suse Linux, Enterprise Edition version 11 SP 0.

Oracle datapump, as mentioned, is a replacement for the old `exp` and `imp` utilities. It comes with numerous benefits, and a couple of minor niggles! All will be revealed.

Internally, datapump uses a couple of PL/SQL packages:

- DBMS_DATAPUMP
- DBMS_METADATA

Normally, you won't need to bother with these - for taking _logical_ backups of the database, but you can, if you wish, call them explicitly. Doing so is beyond the scope of this article - as they say - but if you wish to find out more, have a look in the _Oracle Database PL/SQL Packages and Types Reference Guide_ for more details.

Datapump has two modes:

- Command line
- Interactive

The former is similar to using the old `exp` or `imp` utilities, while the latter allows you to connect to long running datapump jobs, and enquire as to progress and add files or similar. This will be discussed later, but for the rest of us Luddites, command line mode will probably be most familiar to start with.

Before we start, we need to make sure we have a couple of things (technical term) set up.

> Please note. In the following, some Linux commands need to be executed as the root user, these are prefixed with a '#'. All other commands are prefixed by a '$' prompt, and these are executed as the oracle user.

## Prerequisites

When you come to use Datapump utilities, you need to have a pre-existing Oracle Directory within the database being exported or imported. This directory object tells datapump - where to write or read dump files to/from. By default, every database created has a directory already set up for datapump to use. It is called `DATA_PUMP_DIR` and defaults to the location $ORACLE_HOME/rdbms/log.

```sql
SQL> !echo $ORACLE_HOME
/srv/oracle/product/11gR1/db/

SQL> select owner, directory_name, directory_path
  2  from dba_directories
  3  where directory_name = 'DATA_PUMP_DIR';

OWNER   DIRECTORY_NAME  DIRECTORY_PATH
------ -------------- ----------------------------------------
SYS     DATA_PUMP_DIR   /srv/oracle/product/11gR1/db//rdbms/log/
```

This isn't normally the best location, so you have a choice of amending (ie dropping and recreating) the current one, or creating a new one for our own use. I find it best to create a new one:

```sql
SQL> connect / as sysdba
Connected.

SQL> create directory my_datapump_dir as '/srv/oracle/datapump';
Directory created.
```

The location pointed to need not exist when the above command is executed, but it must exist when you attempt to use it and the oracle user must be able to read from and write to the specified location.

```bash
# mkdir /srv/oracle/datapump
# chown oracle:dba /srv/oracle/datapump/
# ls -l /srv/oracle
...
drwxr-xr-x  2 oracle dba   4096 Aug 29 15:03 datapump
...
```

If you only ever intend to run datapump jobs as the SYSDBA enabled users, then this is all we need. However, if you intend to set up another user for this purpose, the following needs to be carried out or the user in question won't be able to run the datapump utilities.

```sql
SQL> create user datapump_admin identified by secret
  2  default tablespace users 
  3  quota unlimited on users;
User created.

SQL> grant create table to datapump_admin;
Grant succeeded.

SQL> grant datapump_exp_full_database to datapump_admin;
Grant succeeded.

SQL> grant datapump_imp_full_database to datapump_admin;
Grant succeeded.

SQL> grant read, write on directory my_datapump_dir to datapump_admin;
Grant succeeded.
```

That's it, we are ready to do some exporting - new style! In case you were wondering, we need the create table privilege because datapump utilities need to create a table for each job executed. More on this later.

## Exporting

This article concentrates mainly on the exporting of data from a database using the `expdp` utility which replaces the old `exp` utility we know and love so much!

### Exporting to a Lower Version of Oracle

There is a nice new parameter that allows easy exporting of data from a higher version of Oracle so that it can be imported into a lower version. It is not used in the details below, but just in case you ever need to export from 12.2 and import into 11.2.0.4, for example, then the `VERSION` parameter will be your best friend.

```bash
$ expdp ..... VERSION=11.2
```
It can, of course, be added to a parameter file too. Details below on the use of parameter files.

### Export Full Database

The command to export a full database is as follows:

```bash
 $ expdp datapump_admin/secret directory=my_datapump_dir dumpfile=full.dmp logfile=full.exp.log full=y
```

However, we can put these parameters into a parameter file - just like when we used `exp`!

```bash
$ cat fulldp.par
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
dumpfile=full.dmp 
logfile=full.exp.log 
full=y
```

If you omit the password from the `userid` parameter, `expdp` and `impdp` will prompt you.

Running a full export is now a simple matter of:
```bash
$ expdp parfile=fulldp.par
```
What happens next is that a pile of "stuff" scrolls up the screen, some of which is useful, some not so. Here is an excerpt with the good bits highlighted:

```text
Export: Release 11.2.0.2.0 - Production on Thu Aug 29 15:20:12 2013
Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.
Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production
With the Partitioning option

Starting **"DATAPUMP_ADMIN"."SYS_EXPORT_FULL_01"**:  datapump_admin/******** parfile=fulldp.par 

Estimate in progress using BLOCKS method...
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
**Total estimation using BLOCKS method: 13.75 MB**

Processing object type DATABASE_EXPORT/TABLESPACE
Processing object type DATABASE_EXPORT/PASSWORD_VERIFY_FUNCTION
Processing object type DATABASE_EXPORT/PROFILE
Processing object type DATABASE_EXPORT/SYS_USER/USER
...
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
...
```

The job name created to carry out the export is DATAPUMP_ADMIN.SYS_EXPORT_FULL_01. Other jobs will have a different numeric suffix to keep them unique. If something goes wrong with the export, we will need that job name to allow us to fix things. The job, while it is in progress, also creates a table within the schema specified in the `userid` parameter. That table's name is also called SYS_EXPORT_FULL_01.

We can also see that an estimate of the amount of disc space we will need in the location where the dump file is being written to. In my case, 13.75 Mb (It's not a big database!) is required.

Then we get a list of the objects being exported as they occur. Nothing much here that's new, it's very similar to that output by a full `exp` in the old days!

Of course, like many things, it doesn't always go to plan:

```text
ORA-39171: Job is experiencing a resumable wait.
ORA-01691: unable to extend lob segment DATAPUMP_ADMIN.SYS_LOB0000015147C00045$$ by 128 in tablespace USERS
```

This LOB is part of the above mentioned table. `Expdp` will sit there (for two hours by default) until I fix the problem. I need to use SQL*Plus (or Toad etc) to fix the underlying problem with space, then `expdp` can continue.

```sql
SQL> select file_id, bytes/1024/1024 as mb
  2  from dba_data_files
  3  where tablespace_name = 'USERS';

   FILE_ID	   MB
---------- ----------
	 6	   10

SQL> alter database datafile 6 resize 50m;
Database altered.
```

As soon as the extra space is added, the datapump job resumes automatically. There is no need to tell it to continue. The screen output starts scrolling again:

```text
...
. . exported "SYSTEM"."REPCAT$_USER_PARM_VALUES"             0 KB       0 rows
. . exported "SYSTEM"."SQLPLUS_PRODUCT_PROFILE"              0 KB       0 rows
**Master table "DATAPUMP_ADMIN"."SYS_EXPORT_FULL_01" successfully loaded/unloaded**
******************************************************************************
Dump file set for DATAPUMP_ADMIN.SYS_EXPORT_FULL_01 is:
  /srv/oracle/datapump/full.dmp
Job "DATAPUMP_ADMIN"."SYS_EXPORT_FULL_01" **completed with 5 error(s)** at 15:46:57
```

The message about the master table means that the table has now been dropped as the datapump job completed. The final message, indicating a number of errors can be quite threatening, but is simply the number of times that the utility output a message to the screen (and log file) telling you that there is a problem:

```text
ORA-39171: Job is experiencing a resumable wait.
ORA-01691: unable to extend lob segment DATAPUMP_ADMIN.SYS_LOB0000015147C00045$$ by 128 in tablespace USERS
ORA-39171: Job is experiencing a resumable wait.
ORA-01691: unable to extend lob segment DATAPUMP_ADMIN.SYS_LOB0000015147C00045$$ by 128 in tablespace USERS
ORA-39171: Job is experiencing a resumable wait.
ORA-01691: unable to extend lob segment DATAPUMP_ADMIN.SYS_LOB0000015147C00045$$ by 128 in tablespace USERS
ORA-39171: Job is experiencing a resumable wait.
ORA-01691: unable to extend lob segment DATAPUMP_ADMIN.SYS_LOB0000015147C00045$$ by 128 in tablespace USERS
ORA-39171: Job is experiencing a resumable wait.
ORA-01691: unable to extend lob segment DATAPUMP_ADMIN.SYS_LOB0000015147C00045$$ by 128 in tablespace USERS
```

As you can see, 5 "errors" that are not really errors, they are merely hints that something needed attending to a bit quicker than I managed!

The full log file for the job is created in the output area, as is the dump file itself.

```bash
$ ls -l /srv/oracle/datapump/

total 18360
-rw-r----- 1 oracle users 18751488 2013-08-29 15:46 full.dmp
-rw-r--r-- 1 oracle users    23410 2013-08-29 15:46 full.exp.log
```

One thing that a number of DBAs will be miffed at, you _cannot_ perform compression of the dump file "on the fly" as we used to do in the old days of exp:

```bash
$ mknop exp.pipe p
$ cat exp.pipe | gzip -9 - > full.dmp.gz &
$ exp ... file=exp.pipe log=full.log ....
```

This is no longer allowed, but `expdp` does have a `compression` parameter however, you need to have paid extra for the _Advanced Compression Option_. Not good! You _are_ allowed, without extra costs, to compress the metadata only when exporting. Thanks Oracle. Add the following to the parameter file:

```text
compression=metadata_only
```

The default is `compression=none`.

And, also, if you run the same export again, then `expdp` will crash out because it doesn't like overwriting files that already exist.

```bash
$ expdp parfile=fulldp.par 
```
```text
Export: Release 11.2.0.2.0 - Production on Thu Aug 29 15:59:29 2013
Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.
Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production
With the Partitioning option
ORA-39001: invalid argument value
ORA-39000: bad dump file specification
ORA-31641: unable to create dump file "/srv/oracle/datapump/full.dmp"
ORA-27038: created file already exists
Additional information: 1
```

There are two ways to avoid this issue:

- Don't put the file names in the parameter file. Take them out and specify them on the command line, explicitly:
    ```bash
    $ cat fulldp.par
    ```
    ```text
    userid=datapump_admin/secret 
    directory=my_datapump_dir 
    full=y
    ```
    ```bash
    $ expdp parfile=fulldp.par dumpfile=full.\`date +%Y%m%d\`.dmp logfile=full.\`date +%Y%m%d\`.exp.log ...
    ```

- Add the following to the parameter file (or command line):
    
    ```text
    reuse_dumpfiles=y
    ```

    The default is not to reuse the existing dump files.

I find that putting all the common parameters into the parameter file, while keeping the changeable ones on the command line is probably the best idea.

What about the old `consistent=y` parameter? Can't you do that with expdp? Well, yes, you can. Since 11gR2 you can anyway. If you add the legacy parameters to the parameter file, or command line as follows:

```text
consistent=y
```

Then `expdp` will notice that you are a Luddite like me, and tell you what to do next time in order to get a proper `expdp` consistent export. As follows:

```text
Export: Release 11.2.0.2.0 - Production on **Fri Aug 30 17:05:33 2013**
...
**Legacy Mode Active due to the following parameters:**
Legacy Mode **Parameter: "consistent=TRUE"** Location: Parameter File, **Replaced with: "flashback_time=TO_TIMESTAMP('2013-08-30 17:05:33', 'YYYY-MM-DD HH24:MI:SS')"**
**Legacy Mode has set reuse_dumpfiles=true parameter.**
Starting "DATAPUMP_ADMIN"."SYS_EXPORT_FULL_01":  datapump_admin/******** parfile=fulldp.par 
...
```

Note the time that the export started, and note how the `consistent` parameter was converted to a `expdp` `flashback_time` parameter set to the same time as the start of the job. That's how to get a consistent export. You can also use the `flashback_scn` parameter if yo happen to knwo the desired SCN of course.

Note also that legacy mode turns on, automatically, the ability to overwrite existing dump files, even if you don't specify it in the parameter file or command line.

> You can, if you wish, add the following to your parameter files, or the command line, according to what you are using, to get an export equivalent to an old `consistent=y` one from `exp`:
> 
> ```text
> flashback_time=to_timestamp(sysdate)
> ```
> or:
> ```text
> flashback_time=systimestamp
> ```

### Export Schemas

Exporting a schema, or more than one, is equally as simple. Simply specify the `schemas=` parameter in your parameter file or on the command line:

```bash
$ cat user_norman.par
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
schemas=norman
reuse_dumpfiles=y
```
```bash
$ expdp parfile=user_norman.par dumpfile=norman.dmp logfile=norman.exp.log
```

If you have more than one schema to export, simply specify them all with commas between. For example, you might have the following in a parameter file (or on the command line):

```text
schemas=barney,fred,wilma,betty,bambam,dino
```

The output is similar to that for the full database:

```text
Export: Release 11.2.0.2.0 - Production on Thu Aug 29 16:39:30 2013
...
Starting **"DATAPUMP_ADMIN"."SYS_EXPORT_SCHEMA_01"**:  datapump_admin/******** parfile=user_norman.par 

Estimate in progress using BLOCKS method...
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 64 KB

Processing object type SCHEMA_EXPORT/USER
...
Processing object type SCHEMA_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
. . exported "NORMAN"."NORM"                             5.023 KB       4 rows
Master table "DATAPUMP_ADMIN"."SYS_EXPORT_SCHEMA_01" successfully loaded/unloaded
******************************************************************************
Dump file set for DATAPUMP_ADMIN.SYS_EXPORT_SCHEMA_01 is:
  /srv/oracle/datapump/norman.01.dmp
Job "DATAPUMP_ADMIN"."SYS_EXPORT_SCHEMA_01" successfully completed at 16:40:04
```

### Export Tablespaces

Exporting a tablespace, or more than one, is just as simple as exporting schemas. Simply specify the `tablespaces=` parameter in your parameter file or on the command line:

```bash
$ cat user_tablespace.par
```
```text
userid=datapump_admin/secret
directory=my_datapump_dir
dumpfile=users_ts.dmp
logfile=users_ts.exp.log      
tablespaces=users
reuse_dumpfiles=y
```
```bash
$ expdp parfile=user_tablespace.par
```

This time, I don't care about having a unique name, so I've specified the `logfile` and `dumpfile` parameters within the parameter file.

If you have more than one tablespace to export, simply specify them all with commas between. For example, you might have the following in a parameter file (or on the command line):

```text
tablespaces=bedrock_data,bedrock_index
```

The output is similar to that for the full database:

```bash
$ expdp parfile=user_tablespace.par 
```
```text
Export: Release 11.2.0.2.0 - Production on Thu Aug 29 16:45:01 2013
...
Starting **"DATAPUMP_ADMIN"."SYS_EXPORT_TABLESPACE_01"**:  datapump_admin/******** parfile=user_tablespace.par 

Estimate in progress using BLOCKS method...
Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 128 KB

Processing object type TABLE_EXPORT/TABLE/TABLE
...
Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
. . exported "APP_OWNER"."TEST"                          5.031 KB       5 rows
. . exported "NORMAN"."NORM"                             5.023 KB       4 rows
Master table "DATAPUMP_ADMIN"."SYS_EXPORT_TABLESPACE_01" successfully loaded/unloaded
******************************************************************************
Dump file set for DATAPUMP_ADMIN.SYS_EXPORT_TABLESPACE_01 is:
  /srv/oracle/datapump/users_ts.dmp
Job "DATAPUMP_ADMIN"."SYS_EXPORT_TABLESPACE_01" successfully completed at 16:45:27
```

### Export Tables

And finally, for now, exporting a list of tables is simple too. Once again, and as with `exp`, you are best to fill a parameter file with the required tables.

```bash
$ cat tables.par 
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
dumpfile=tables.dmp 
logfile=tables.exp.log 
tables=norman.norm,app_owner.test
reuse_dumpfiles=y
```

As before, I'm only interested in these particular tables, so I name them in the parameter file. Also, `dumpfile` and `logfile` are in there too.

You should be quite familiar with the output by now:

```bash
expdp parfile=tables.par 
```
```text
Export: Release 11.2.0.2.0 - Production on Thu Aug 29 16:49:05 2013
...
Starting **"DATAPUMP_ADMIN"."SYS_EXPORT_TABLE_01"**:  datapump_admin/******** parfile=tables.par 

Estimate in progress using BLOCKS method...
Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 128 KB

Processing object type TABLE_EXPORT/TABLE/TABLE
Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
. . exported "APP_OWNER"."TEST"                          5.031 KB       5 rows
. . exported "NORMAN"."NORM"                             5.023 KB       4 rows
Master table "DATAPUMP_ADMIN"."SYS_EXPORT_TABLE_01" successfully loaded/unloaded
******************************************************************************
Dump file set for DATAPUMP_ADMIN.SYS_EXPORT_TABLE_01 is:
  /srv/oracle/datapump/tables.dmp
Job "DATAPUMP_ADMIN"."SYS_EXPORT_TABLE_01" successfully completed at 16:49:12
```
## Points to Note

### Job names.

The job names created for an `expdp` or `impdp` job are made up as follows:

schema_name + "." + "SYS_" + "EXPORT_" or "IMPORT_" + Level + "_" + Unique Identifier.

`Expdp` uses "EXPORT" while `impdp` uses "IMPORT", for obvious reasons. The level part is one of:

- FULL
- SCHEMA
- TABLESPACE
- TABLE

The unique identifier is simply a numeric suffix, starting at 01 and increasing for each _concurrent_ datapump job at that level.

So, a job running under the schema of datapump_admin, exporting a schema level dump, would have the full job name of:

DATAPUMP_ADMIN.SYS_EXPORT_SCHEMA_01

While the same user, exporting a full database would have the job name of

DATAPUMP_ADMIN.SYS_EXPORT_FULL_01

### Tables created for jobs.

As mentioned above, datapump jobs create a table in the default tablespace of the user running the utility. The table names are exactly the same as the running job names.

When the job completes, the tables are dropped.

### Estimating space requirements.

As seen above, `expdp` produces an estimate of the space it will need to carry out the requested export. However, it might be nice to have this information well in advance of running the export - so that you can be sure that the export will work without problems. This can be done:

```bash
$ expdp datapump_admin/secret full=y estimate_only=y
```
```text
Export: Release 11.2.0.2.0 - Production on Thu Aug 29 16:27:26 2013
...
Starting "DATAPUMP_ADMIN"."SYS_EXPORT_FULL_01":  datapump_admin/******** full=y estimate_only=y 
Estimate in progress using BLOCKS method...
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
...
.  estimated "SYSTEM"."REPCAT$_TEMPLATE_TARGETS"             0 KB
.  estimated "SYSTEM"."REPCAT$_USER_AUTHORIZATIONS"          0 KB
.  estimated "SYSTEM"."REPCAT$_USER_PARM_VALUES"             0 KB
.  estimated "SYSTEM"."SQLPLUS_PRODUCT_PROFILE"              0 KB
**Total estimation using BLOCKS method: 13.75 MB**
Job "DATAPUMP_ADMIN"."SYS_EXPORT_FULL_01" successfully completed at 16:27:39
```
Now, armed with the above information, you can make sure that the destination for the dump file(s) has enough free space. Don't forget, there will be a log file as well.

The [next article](/posts/2013/09/introduction-to-oracle-datapump-part-2) in this short series will feature the corresponding imports using `impdp`.

## Exporting - Cheat Sheet

The following is a list of "cheats" - basically, a list of the parameters you would find useful in doing a quick export at any level from full down to individual tables. I have listed each one in the form of a parameter file - for ease of copy and paste, which you are free to do by the way, assuming you find it interesting and/or useful!

The following assumes you have set up a suitable Oracle Directory object within the database being exported, however, the first part of the cheat sheet summarises the commands required to create one, and a suitably endowed user to carry out the exports.

All the following default to consistent exports, there's really no reason why an export should be taken any other way!

### Create an Oracle Directory

The following is executed in SQL*Plus, or similar, as a SYSDBA enabled user, or SYS:

```sql
create directory **my_datapump_dir** as '/your/required/location';
```

The following is executed as root:

```bash
mkdir -p /your/required/location
chown oracle:dba /your/required/location
```

### Create a Datapump User Account and Privileges

The following is executed in SQL*Plus, or similar, as a SYSDBA enabled user, or SYS:

```sql
create user datapump_admin identified by secret_password
default tablespace users 
quota unlimited on users;

grant create table to datapump_admin;
grant datapump_exp_full_database to datapump_admin;
grant datapump_imp_full_database to datapump_admin;
grant read, write on directory **my_datapump_dir** to datapump_admin;
```

### Full, Consistent Export

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=full.dmp 
logfile=full.exp.log 
**full=y**
reuse_dumpfiles=y
flashback_time="TO_TIMESTAMP('your_date_time_here', 'YYYY-MM-DD HH24:MI:SS')"
```

You may wish to use `consistent=y` rather than the `flashback_time`, it will default to the timestamp of the start of the `expdp` job.

### Consistent Schema(s) Export

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=schema.dmp 
logfile=schema.exp.log 
**schemas=user_a,user_b**
reuse_dumpfiles=y
flashback_time="TO_TIMESTAMP('your_date_time_here', 'YYYY-MM-DD HH24:MI:SS')"
```

You may wish to use `consistent=y` rather than the `flashback_time`, it will default to the timestamp of the start of the `expdp` job.

### Consistent Tablespace(s) Export

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=tablespaces.dmp 
logfile=tablespaces.exp.log 
**tablespaces=ts_a,ts_b**
reuse_dumpfiles=y
flashback_time="TO_TIMESTAMP('your_date_time_here', 'YYYY-MM-DD HH24:MI:SS')"
```

You may wish to use `consistent=y` rather than the `flashback_time`, it will default to the timestamp of the start of the `expdp` job.

### Consistent Table(s) Export

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=tables.dmp 
logfile=tables.exp.log 
**tables=user.table_a,user.table_b**
reuse_dumpfiles=y
flashback_time="TO_TIMESTAMP('your_date_time_here', 'YYYY-MM-DD HH24:MI:SS')"
```

You may wish to use `consistent=y` rather than the `flashback_time`, it will default to the timestamp of the start of the `expdp` job.

### Adding Compression

By default there is no compression. You add it as follows in 10g and higher _with no further options purchased_:

```text
compression=metadata_only
```

Or off with this:

```text
compression=none
```

### Adding Even More Compression

If you have purchased a license for Oracle's _Advanced Compression Option_, in Oracle 11g and higher, then you have the options of adding compression as follows:

Nothing is compressed:

```text
compression=none
```

Only the metadata is to be compressed:

```text
compression=metadata_only
```

Compress the data only:

```text
compression=none
```

Compress the metadata and the data:

```text
compression=all
```

### Even More Compression, in 12c

From 12c, a new parameter named `compression_algorithm` allows you to specify which level of compression you would like:

```text
compression_algorithm=basic

compression_algorithm=low

compression_algorithm=medium

compression_algorithm=high
```

These options may well incur extra CPU costs as the data are required to be compressed for export and uncompressed on import.

### Adding Encryption

If you have Enterprise Edition _and_ have paid the extra cost license fee for the _Oracle Advanced Security_ option, then you can encrypt the data in the dump files.

Encrypt the metadata and the data:

```text
encryption=all
```

Encrypt the data only:

```text
encryption=data_only
```

Encrypt just the metadata:

```text
encryption=data_only
```

No encryption (the default):

```text
encryption=none
```

Encrypt the data in columns that are defined as encrypted in the database:

```text
encryption=encrypted_columns_only
```
