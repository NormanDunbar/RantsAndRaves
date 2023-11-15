---
title: "Beware of the Silent Database Killer!"
date: "2014-10-25"
categories: 
  - "oracle"
---

There is, out there in Oracle Land, a silent database killer. You never know when it will strike and it affects all databases right up to and including 12c. When it strikes, it does so silently, there is no evidence of its passing, until it is far too late.

### What is This Killer?

The database killer is any code which runs in `NOLOGGING` or `UNRECOVERABLE` or, in some cases prior to 11g, `DIRECT PATH` loads.

The following are examples of these sorts of commands:

- `INSERT /\*+ Append \*/ INTO ...` (**11g is not affected by this.**)
- Any DML after `ALTER .... UNRECOVERABLE;`
- Any DML after `ALTER .... NOLOGGING;`
- `CREATE ... UNRECOVERABLE;`
- `CREATE ... NOLOGGING;`

Alternatively, using SQL Loader with any of the following parameters in the control file:

- `UNRECOVERABLE`
- `OPTIONS(DIRECT=TRUE)`

Or, running SQL Loader with the following command line option:

- `sqlldr direct=true`

### Useful MOS Documents

The following documents will prove very useful if you are affected by this silent killer.

- 290161.1 : The Gains and Pains of Nologging Operations.
- 269274.1 : Check For Logging / Nologging On DB Object(s).
- 751249.1 : Dbv-111 Ora-1219 Sys.X$Dbms_dbverify. (Or, in English, what to do when DBV on a standby data file throws an OCI error ORA-01219.)
- 472231.1 : How to identify all the Corrupted Objects in the Database with RMAN.
- 605234.1 : How to Copy ASM datafiles from Primary to Standby Database on ASM using RMAN.

### Prevention is Better Than Cure

The following command _must_ be executed on the primary and standby databases, if you have any.

```sql
alter database force logging;
```

Downtime is not required and if there is any of the `NOLOGGING` work in progress, the `ALTER DATABASE` will hang until such time as the `NOLOGGING` work completes. A message will be logged to the alert.log advising you of this.

The command executes quite happily on a physical standby database without the need to cancel recovery.

The result of the above command is that any attempt by an SQL operation to attempt a `NOLOGGING` operation will be ignored, and full logging will take place, resulting in the safety of your data and the continuing viability of the standby database(s).

### How Can I Tell if I'm Infected?

The following SQL statement will hopefully return no rows. However, if there are any rows returned, those are the data files that have been updated at _some point in the past_, with a `NOLOGGING` operation of some kind. They are sitting there, silently, waiting for an excuse to kill your database.

```sql
alter session set nls_date_format = 'dd/mm/yyyy hh24:mi:ss';

select file#, name, unrecoverable_change#, unrecoverable_time
from v$datafile
where unrecoverable_change# <> 0
order by file#;

The `V$DATAFILE` view only holds the most recent unrecoverable change details against each datafile. There may have been others in the past, but the one you see is the most recent.

No rows selected is good, but sometimes, you may see something like this:

FILE# NAME                                UNRECOVERABLE_CHANGE# UNRECOVERABLE_TIME
----- ----------------------------------- --------------------- -------------------
    6 +DATA/orcl/data/users.293.825959933        10917504423565 13/10/2014 15:00:34
```

As usual, database names etc have been changed to protect the innocent!

It makes no difference if the files are in ASM, as in this example, or on file systems, the problem is the same and needs to be attended to _urgently_.

If you find any data files with an unrecoverable change, as above, then the most obvious thing to do is immediately take a full backup of the database (or just the affected data files) because if you have to restore and recover the affected data files, that is the time when the corruptions are introduced into the primary database. The standby database, on the other hand, is already dead - it just doesn't know it yet.

To check if your primary database is also dead, and to save you backing up a potentially dead primary, you can run the following in RMAN - there's no need to connect with the catalog, if you use one:

```sql
backup validate check logical datafile 6;
```

If you see a non-zero number in the section entitled "Marked Corrupt" then it means that at some point since the unrecoverable change was applied to the primary database, it has been restored and recovered, and the data that was loaded is now missing. This database is mortally wounded - unless you know what data needs to be reloaded _and_ you will need to "uncorrupt" those affected datafiles by restoring or recreating them, and their contents.

If your database has a standby, then the standby is now not viable to be used as a primary. All the data that have been loaded into the primary database using `NOLOGGING`, or similar operations, has _never_ been loaded into the standby database. If you detect any "corrupt" data files on the primary database, as above, with a date & time later than the standby database's creation date & time, then you will need to rebuild or repair the standby database. You can check the `CREATED` column in `V$DATABASE` to determine the standby database's creation details.

You _will_ be able to run a data guard switch over, either manually, or with OEM or DGMGRL, without error. When the current standby comes up as the new primary database, there _will_ be missing data. If the application and/or developers/vendor continues to run `NOLOGGING` data loads, then the amount of data loss simply increases. When you switch back to the old primary at some point, everything will be in a mess - _and you will not know_!

How big a mess?

- There is the data originally loaded into the old primary, that is not present on the old standby - now the new primary - but which is present on the new standby.
- There is also now, the data being loaded into the new primary, that is not being copied to the new standby (the old primary) - so both databases are missing some data.

If you switch back and forthe a few times, the mess just keeps getting messier!

### So What's Going On?

Some vendors and/or developers, and possibly even the odd DBA, have read in the manuals that using `NOLOGGING`, `UNRECOVERABLE` or `DIRECT PATH` operations can "save time" or "improve performance" by not logging the data changes to the redo logs for the actual data. Changes to the data dictionary _will_ still be logged and transferred to the standby databases.

These same people, however, appear to completely ignore the documentation where it says that "whenever you use a `NOLOGGING` operation, you must take a full database backup" immediately afterwards.

This is a problem. If you load a table with millions of rows in this manner, the table may extend by adding extents. These new extents will be recorded in the dictionary and will match the actual data usage of the table. The redo logs, however, will only record the changes to the dictionary. The standby database will update its dictionary with the details, but the table on the standby will not be updated with either the data or the new extents - it will not change it's row or extent count at all.

The manual states that these operations should _only be used on objects that are not required to be recovered_. You might have a table that is simply used as the temporary source for a data load before it is transformed and loaded into the correct, final tables. You don't care about recovering the data when it was temporary to begin with. However, the database is not a mind reader and doesn't know when you create a temporary table, use it, and the perhaps drop it, that that object was never required to be recovered. The standby datafiles will be flagged corrupt and the primary datafiles will still log an unrecoverable change.

As long as the primary database continues to run happily, the data thus loaded, can be manipulated at will in the normal manner.

If the primary database is _ever_ restored and recovered using the archived redo logs created by the data loads, no errors or warnings will be displayed, but the previously loaded data _will not_ be present afterwards. Attempting to access the data after a restore and recover will result in an error similar to the following:

```text
ORA-01578: ORACLE data block corrupted (file # 6, block # 139)
ORA-01110: data file 6: '+DATA/orcl/data/users.293.825959933'
ORA-26040: Data block was loaded using the NOLOGGING option
```

A similarly nasty error will occur when the current standby database is opened read only, or switched over to become the new primary, but remember, the error only becomes apparent when the data are accessed in some way - this is really nasty!

### How to Determine if You Are Affected

On the primary database, you can run the `dbv` utility against all the data files. Use the userid parameter if the data files live in ASM:

```bash
dbv file='+DATA/orcl/data/users.293.825959933' userid=sys/password

DBVERIFY: Release 11.2.0.4.0 - Production on Sat Oct 25 15:03:44 2014
Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = +DATA/orcl/data/users.293.825959933

DBV-00201: Block, DBA 25165963, marked corrupt for invalid redo application
DBV-00201: Block, DBA 25165964, marked corrupt for invalid redo application
DBV-00201: Block, DBA 25165965, marked corrupt for invalid redo application
DBV-00201: Block, DBA 25165966, marked corrupt for invalid redo application

DBVERIFY - Verification complete

Total Pages Examined         : 16
Total Pages Processed (Data) : 8
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 0
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 7
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 1
Total Pages Marked Corrupt   : 4
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 0 (0.0)
```

The above shows that 4 pages (aka blocks) are marked corrupt. This is what you will see on a primary that has been restored and recovered. You will not see any corruption if the primary has not been restored and recovered, the data are still present in that case.

Checking the standby is equally as simple as non-corrupt data files will happily `dbv`. However, any data file that is corrupted will cause `dbv` to abort with then following errors:

```text
DBV-00111: OCI failure (4157) (ORA-00604: error occurred at recursive SQL level 1
ORA-01219: database not open: queries allowed on fixed tables/views only
ORA-06512: at "SYS.X$DBMS_DBVERIFY", line 22
```

If you see this, then your standby is invariable corrupt, however, to be absolutely certain, and you can do this on the primary as well - it's quicker than `dbv` by the way - use RMAN to do a pretend backup with a `CHECK LOGICAL` clause:

```sql
backup validate check logical datafile 6;
```

**Note:** From 11g onwards, the backup part of the command is not required:

```sql
validate check logical datafile 6;
```

A corrupt data file will throw up results similar to the following:

```text
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
6    OK     4              50           12801           7251912

  File Name: +DATA/orcl/data/users.293.825959933
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              8
  Index      0              0
  Other      0              12742
```

The Marked Corrupt column looks interesting, and shows that this data file, on the standby, is indeed corrupt. The data that should be present, is not and this is what silently makes our standby database completely and utterly useless.

We can check the extent and reasons for the corruption after an RMAN check by reading from `V$DATABASE_BLOCK_CORRUPTION` in SQL\*Plus:

```sql
select \* from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO
---------- ---------- ---------- ------------------ ---------
         6        139          4            7251895 NOLOGGING
```

This shows that there are 4 blocks, beginning at block 139, in data file 6 which have had `NOLOGGING` operations applied. If there are numerous corruptions, the following might be a better advisory query:

```sql
select file#, corruption_type, count(\*)
from v$database_block_corruption
group by file#, corruption_type;

     FILE# CORRUPTIO    COUNT(
---------- --------- ---------
         6 NOLOGGING         1
```

If you need to find out what objects are corrupted, the following might be useful, you will need to plug in the starting block numbers from the query above.

```sql
select owner, segment_type, segment_name
from dba_extents
where file_id = 6
and 139 between block_id and block_id+blocks;

OWNER                SEGMENT_TYPE         SEGMENT_NAME
-------------------- -------------------- --------------------
NORMAN               TABLE                TEST
```

### Rebuilding The Standby Database

If you have detected corruptions then your standby databases are useless and need rebuilding. Oracle advise that only the affected data files need to be rebuilt, which is great if the database is huge and only a subset of the files are corrupt. This is documented in MOS note 605234.1, but I have found that it doesn't appear to work.

> I have, however, worked out the problem and a suitable workaround until the bug gets fixed. I have raised an SR on this matter.
> 
> Basically what happens is that the first `switch datafile to copy` command works correctly. The second one which _should_ switch to the latest datafile copy does not, and switches back to the previous data file, the corrupted one.
> 
> The work around is to carry out the first switch, list the copies of the affected datafile, and delete the copy that was the original corrupt file.

The process starts on the primary with an RMAN backup of a non-corrupted data file. There's no point running this rebuild if the primary data files have been restored and recovered, the data will be missing there too, so check first as advised way back near the start of this article.

```sql
copy datafile '+DATA/orcl/data/users.293.825959933' to '/tmp/users.dbf;
exit
```

This file, `/tmp/users.dbf` should be copied over to a safe place on the standby server. If you use `ftp` then remember to transfer in binary. `Scp` or `sftp` will only do binary transfers.

Once the file exists on the standby, we can use RMAN to get the file copied into ASM and used by the standby:

```sql
catalog datafilecopy '/tmp/users.dbf';

sql "alter database recover managed standby database cancel";

switch datafile 6 to copy;                  # Now running on the file in /tmp.
backup as copy datafile 10 format '+DATA';  # Get a copy of the /tmp file into ASM.
switch datafile 6 to copy;                  # Should switch to the new file, but doesn't.

--========================================
-- Insert workaround here. See text above.
--========================================

sql "alter database recover managed standby database using current logfile disconnect";
```

As mentioned above, doing all this and running another `validate check logical` command, still shows corruptions on the standby, even though there are none on the primary. This is because the second `switch datafile to copy` command actually switched back to the old corrupted file instead of the new one. To get around this problem, follow these steps:

- Switch to the /tmp copy as above.
- Backup the /tmp file into +DATA as above.
- `Switch datafile n to copy;` again. This is now using the corrupt file again.
- `List copy of datafile n;` will show the /tmp file and the new backup in ASM.
- `Delete copy of datafile n tag "tag for the /tmp copy";` will delete the /tmp file leaving only the new file in ASM - the one we want.
- `Switch datafile n to copy;` yet again! This is now correctly using the latest uncorrupted data file in ASM.
- `List copy of datafile n;` will show the old corrupt file as the sole remaining copy.
- `Delete copy of datafile n;` will get rid of the corrupt file.

So, a bit of mucking about to get the problem worked around, and you may need to do a lot more mucking about if you have other copies of the same datafile in RMAN, to get the correct new file switched into use and to get rid of the unwanted corrupt file. I didn't have this problem as my own backups are done as backupsets, not copies.

Don't let a silent killer destroy your databases, keep `force loggging` turned on - it's sometimes the only defence against half informed vendors, developers and the odd DBA! ;-)
