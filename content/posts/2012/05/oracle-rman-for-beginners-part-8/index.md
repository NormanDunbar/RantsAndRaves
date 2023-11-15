---
title: "Oracle RMAN for Beginners - Part 8"
date: "2012-05-15"
categories: 
  - "oracle"
---

In the [previous instalment](/posts/2012/03/oracle-rman-for-beginners-part-7/ "Oracle RMAN for Beginners – Part 7") of this exciting series, we completed the full (complete) recovery of the database, tablespaces and data files, as well as looking at recovering individual blocks.

In this article, we will perform incomplete recovery where we restore and recover the databases to a specific point in time that is previous to "now".

> Note that in the following examples I'm using the until time option and specifying a date and time. You can restore to a particular SCN, or archive log number etc. See the manual for all the options.

It is possible that data will be lost with this kind of recovery. It is not possible to perform incomplete recovery on tablespaces or data files, you are limited to restoring and recovering the entire database. Although, this is not quite true, read on ...

## Incomplete Recovery of the Database

As the whole database is being restored, the `SYSTEM` and `UNDO` tablespaces will be restored and recovered, so the database will need to be in a `mounted` state. You cannot carry out incomplete recovery on an open database.

First of all, let's add a new row to our test table which we use to show progress etc.

```sql
SQL> alter session set nls_date_format='dd/mm/yyyy hh24:mi:ss';
Session altered.

SQL> insert into test values (sysdate);
1 row created.

SQL> commit;
Commit complete.

SQL> select * from test order by 1;

A
-------------------
06/02/2012 20:25:11
16/02/2012 17:26:20
17/02/2012 15:03:42
17/02/2012 15:54:49
07/03/2012 11:21:53
19/03/2012 20:58:18
15/05/2012 13:44:29

7 rows selected.
```

Now, restore and recover the database back to a time that is about 15 minutes ago. The first stage is to get the database into a `mount` state.

```bash
rman target /

RMAN> shutdown;

database closed
database dismounted
Oracle instance shut down

RMAN> startup mount

connected to target database (not started)
Oracle instance started
database mounted

Total System Global Area     768331776 bytes

Fixed Size                     2230360 bytes
Variable Size                213911464 bytes
Database Buffers             549453824 bytes
Redo Buffers                   2736128 bytes
```

Next, we restore the database to our chosen time.

```sql
RMAN> restore database until time "to_date('15/05/2012 13:30:00','dd/mm/yyyy hh24:mi:ss')";
```
```text
Starting restore at 15/05/2012 14:08:47
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=19 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00002 to /srv/nffs/oradata/ant12/data/sysaux01.dbf
...
...
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 15/05/2012 14:13:04
```

I've trimmed the usual RMAN verbose output from the above to save you falling asleep reading it! ;-)

Next, we need to recover the database to the same time.
```sql
RMAN> recover database until time  "to_date('15/05/2012 13:30:00','dd/mm/yyyy hh24:mi:ss')";
```
```text
Starting recover at 15/05/2012 14:16:35
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 12 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_12_7mwyxlk8_.arc
...
...
media recovery complete, elapsed time: 00:00:36
Finished recover at 15/05/2012 14:17:24
```

Again, I've trimmed the output. RMAN will restore to disc any archived logs it requires to carry out the recovery if it determines that they are not available online.

The final stage is to reopen the database. You must use the resetlogs option because you've carried out an incomplete recovery.

```sql
RMAN> alter database open resetlogs;
database opened
```

So, the restore and recovery went well, did it really work?

```sql
SQL> conn norman/norman
Connected.

SQL> alter session set nls_date_format='dd/mm/yyyy hh24:mi:ss';
Session altered.

SQL>  select * from test;

A
-------------------
07/03/2012 11:21:53
19/03/2012 20:58:18
06/02/2012 20:25:11
16/02/2012 17:26:20
17/02/2012 15:03:42
17/02/2012 15:54:49

6 rows selected.
```

So, our most recent row in the table is no longer present, we now have 6 rows instead of the 7 that we had at the start of this recovery. The database has been restored back in time.

You will note that in both the `restore database` and the `recover database` commands, I had to specify the same time. There is a short cut that you can use to save typing.

```sql
RMAN> run {
2> set until time = "to_date('15/05/2012 13:30:00','dd/mm/yyyy hh24:mi:ss')";
3> restore database;
4> recover database;
5> }
```
```text
executing command: SET until clause

Starting restore at 15/05/2012 14:02:23
using channel ORA_DISK_1
...
...
Finished restore at 15/05/2012 14:02:24

Starting recover at 15/05/2012 14:02:24
using channel ORA_DISK_1
...
...
media recovery complete, elapsed time: 00:00:02

Finished recover at 15/05/2012 14:02:26
```

Now you can open the database in `resetlogs` mode as above.

## Restoring a Database Through a Resetlogs

In the past it wasn't possible to restore a database that had been opened with `resetlogs` specified. RMAN used to treat this as a new incarnation of the database, so you had to make sure you took a fresh backup as soon as the database was opened.

RMAN in 11g is much more forgiving and it is now possible to restore through a `resetlogs`. You do this by restoring a backup from the previous incarnation and do a recover. If you watch the recovery phase carefully, you will see the archived logs files being applied from before and after the `resetlogs`.

The following assume that you haven't taken a backup of the new incarnation yet. If you have, you must restore the controlfile from a named backup as opposed to letting RMAN work it out.

The first step is to restore the controlfile.

```sql
RMAN> shutdown immediate;

database closed
database dismounted
Oracle instance shut down
```

```sql
RMAN> startup nomount;

connected to target database (not started)
Oracle instance started

Total System Global Area     768331776 bytes

Fixed Size                     2230360 bytes
Variable Size                213911464 bytes
Database Buffers             549453824 bytes
Redo Buffers                   2736128 bytes
```

```sql
RMAN> restore controlfile from autobackup;

Starting restore at 15/05/2012 15:12:57
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=17 device type=DISK
...
...
Finished restore at 15/05/2012 15:13:03
```

```sql
RMAN> alter database mount;

database mounted
released channel: ORA_DISK_1
```

Now that we are on an old controlfile and the database is mounted, we can restore and recover the database in the normal manner.

```sql
RMAN> restore database;
```
```text
Starting restore at 15/05/2012 15:15:06
Starting implicit crosscheck backup at 15/05/2012 15:15:06
allocated channel: ORA_DISK_1
...
...
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 15/05/2012 15:19:27
```

Now all we have to do is the recover. As mentioned above, watch the thread and sequence numbers closely.

```sql
RMAN> recover database;
```
```text
Starting recover at 15/05/2012 15:19:36
using channel ORA_DISK_1

starting media recovery
...
...
archived log ... thread=1 sequence=8
archived log ... thread=1 sequence=9
...
...
archived log ... thread=1 sequence=20
archived log ... thread=1 sequence=1
media recovery complete, elapsed time: 00:00:52
Finished recover at 15/05/2012 15:20:42
```

You can hopefully see, in the above, that the sequence number dropped from 20 to 1 as we recovered through the resetlogs. Talking of which, we need another one now.

```sql
RMAN> alter database open resetlogs;
database opened
```

## Tablespace Incomplete Recovery

It was never possible to do a tablespace point in time recovery without cloning the database first and using the clone to export the data. However, with 11g it is now possible to perform incomplete recovery on a tablespace (or tablespaces) as RMAN takes care of all the cloning etc.

First of all, create a working area for the clone, as root.

```bash
mkdir /media/oracle_backups/working_aux
chown oracle:oinstall /media/oracle_backups/working_aux/
ls -l /media/oracle_backups/

total 24
drwxr-xr-x 2 oracle oinstall  4096 Feb 17 16:51 ant12
drwx------ 2 root   root     16384 Feb  6 08:07 lost+found
drwxr-xr-x 2 oracle oinstall  4096 May 15 15:34 working_aux
```

Next, check our test table to see how far back we can recover the users tablespace.

```sql
SQL> conn norman/norman
Connected.

SQL> alter session set nls_date_format='dd/mm/yyyy hh24:mi:ss';
Session altered.

SQL> select * from test;

A
-------------------
07/03/2012 11:21:53
19/03/2012 20:58:18
06/02/2012 20:25:11
16/02/2012 17:26:20
17/02/2012 15:03:42
17/02/2012 15:54:49

6 rows selected.
```

_Obviously_ your decision criteria for the actual time to restore back to will be a little more scientific than mine! But I'm going to select a date of just after 16/02/2012 17:26:20, lets call it 16/02/2012 17:30:00.

The first problem we may happen across is whether there are any objects in the tablespace to be restored which have constraints etc in other tablespaces. We check the `SYS.TS_PITR_CHECK`.

```sql
SQL> conn / as sysdba
Connected.

SQL> select * from ts_pitr_check
  2  where (ts1_name = 'USERS' and ts2_name <> 'USERS') or
  3 (ts2_name = 'USERS' and ts1_name <> 'USERS');
no rows selected
```

In this case as we are about to restore the users tablespace, we have to see if there are relationships from users to other tablespaces and also if there are relationships from other tablespaces to users.

There are none, but if there had been, we would either need to temporarily drop those constraints or include the other tablespace(s) in the recovery.

The next problem is those objects that exist in the database now, that didn't at the time of the recovery point in time. Again we can find a list of those by checking `SYS.TS_PITR_OBJECTS_TO_BE_DROPPED` and giving the date and time we intend to use.

```sql
SQL> select tablespace_name, owner, name
  2  from ts_pitr_objects_to_be_dropped
  3  where tablespace_name = 'USERS'
  4  and creation_time > to_date('16/02/2012 17:30:00','dd/mm/yyyy hh24:mi:ss');
no rows selected
```

So far so good. Had there been any objects listed, we would need to run an export of those objects using data pump to preserve them across the recovery.

We are now ready to carry out a point in time restore of the users tablespace to the working area instead of to the normal database area. You _do not_ need to take the tablespace offline, RMAN does that for you.

> Note, in the following command `_recover_` is correct. Do not use `_restore_` as it will not work.

```sql
RMAN> recover  tablespace users until time "to_date('16/02/2012 17:30:00','dd/mm/yyyy hh24:mi:ss')"
2>  auxiliary destination '/media/oracle_backups/working_area';
```
```text
Starting recover at 15/05/2012 15:55:52
using channel ORA_DISK_1

Creating automatic instance, with SID='qvbt'

initialization parameters used for automatic instance:
db_name=NORM
db_unique_name=qvbt_tspitr_NORM
...
...
Removing automatic instance
Automatic instance removed
Finished recover at 15/05/2012 15:58:42
```

You will note that there is a _lot_ of output from RMAN as it builds a clone database and recovers the tablespace to the specified time.

> You cannot recover a tablespace to a time previous to the last resetlogs time. If you try, you will get the following error message:
> 
> RMAN-03002: failure of recover command at 05/15/2012 15:55:53
> RMAN-20207: UNTIL TIME or RECOVERY WINDOW is before RESETLOGS time

After the tablespace has been cloned and recovered, you must back it up and then bring it online.

```sql
RMAN> backup tablespace users;
...
RMAN> sql 'alter tablespace users online';
...
```

> If you do not have an RMAN catalogue in use, you are not permitted to carry out more than one of these tablespace point in time recovery operations because the controlfile no longer knows about the previous incarnations of the recovered tablespace(s).
> 
> This is one of the reasons why you must backup the tablespace immediately after recovering it.
> 
> If, on the other hand, you do use a catalogue, then multiple recoveries of the same tablespace(s) are permitted.
