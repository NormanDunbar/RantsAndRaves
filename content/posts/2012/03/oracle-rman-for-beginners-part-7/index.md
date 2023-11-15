---
title: "Oracle RMAN for Beginners – Part 7"
date: "2012-03-19"
categories: 
  - "oracle"
---

In [part 6](/posts/2012/02/oracle-rman-for-beginners-part-6/ "Oracle RMAN for Beginners – Part 6") of this mini-series, I left you with a backed up database, having using RMAN to take a hot backup. This episode looks at restoring and recovering from hot backups.

The joy of this is that _most of the time_ you don't need to have everyone off of the database twiddling their thumbs while you restore and recover, just anyone in those areas affected.

Restoration and recovery is _full_ in that the database will be completely recovered right up to date after the restore and recovery is finished.

## Recover the Entire Database

To recover the entire database you do actually need to have the database `mounted` so the users will need to be offline. The steps involved are:

Make sure that the database is mounted, not open:

```sql
RMAN> shutdown

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

Restore the database from a suitable dump. RMAN will choose a suitable dump, or dumps, for you in order to reduce the amount of work required:

```sql
RMAN> restore database;
```
```text
Starting restore at 2012/03/19 21:19:35
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=17 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00002 to /srv/nffs/oradata/ant12/data/sysaux01.dbf
channel ORA_DISK_1: restoring datafile 00003 to /srv/nffs/oradata/ant12/data/undotbs01.dbf
...
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 2012/03/19 21:24:02
```

Recover the database next. This will use any available backups of archived logs, the currently online and possibly never backed up archived logs plus the online redo logfiles to bring the database right up to date. Any archived logs that are no longer online will be restored first - which you can see in the following if you looks carefully:

```sql
RMAN> recover database;
```
```text
Starting recover at 2012/03/19 21:28:58
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 12 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_12_7mwyxlk8_.arc
archived log for thread 1 with sequence 13 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_13_7mwzxtxg_.arc
...
channel ORA_DISK_1: starting archived log restore to default destination
channel ORA_DISK_1: restoring archived log
archived log thread=1 sequence=8
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_annnn_TAG20120217T160332_7mwylo3r_.bkp
...
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
...
archived log file name=/srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_9_7ph990ph_.arc thread=1 sequence=9
channel default: deleting archived log(s)
archived log file name=/srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_9_7ph990ph_.arc RECID=77 STAMP=778368544
archived log file name=/srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_10_7ph990r7_.arc thread=1 sequence=10
channel default: deleting archived log(s)
archived log file name=/srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_10_7ph990r7_.arc RECID=76 STAMP=778368544
channel ORA_DISK_1: starting archived log restore to default destination
channel ORA_DISK_1: restoring archived log
archived log thread=1 sequence=11
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_annnn_TAG20120217T160922_7mwyxlvn_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_annnn_TAG20120217T160922_7mwyxlvn_.bkp tag=TAG20120217T160922
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
archived log file name=/srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_11_7ph996wn_.arc thread=1 sequence=11
channel default: deleting archived log(s)
archived log file name=/srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_11_7ph996wn_.arc RECID=78 STAMP=778368551
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_12_7mwyxlk8_.arc thread=1 sequence=12
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_13_7mwzxtxg_.arc thread=1 sequence=13
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_14_7mwzxx1n_.arc thread=1 sequence=14
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_15_7mwzxxkb_.arc thread=1 sequence=15
media recovery complete, elapsed time: 00:00:25
Finished recover at 2012/03/19 21:29:36
```

And finally, open the database:

```sql
RMAN> alter database open;
database opened
```

## Recover Tablespaces

Recovering individual tablespaces is done with the database open. Only the tablespaces to be recovered need to be offline.

However, if the SYSTEM or UNDO tablespaces need to be recovered, the database will need to be `mounted` as you cannot restore and recover those with the database online - for pretty obvious reasons to be honest!

{{< alert theme="danger" >}}
You cannot recover a tablespace that has been dropped. In that situation, you must perform a point in time recovery to just before the tablespace was dropped. The control file doesn't keep details of the dropped tablespace.

Attempting to recover a dropped tablespace will result in **RMAN-20202 Tablespace not found in the recovery catalog** errors.

You can, however, recover a tablespace where one or more of its datafiles have become corrupted or have been removed by nefarious means.
{{< /alert >}}

The following example shows the users tablespace being restored from a backup and recovered completely up to date.

First of all, in an SQL*Plus session, add an up to date record to a test table in the tablespace to be restored and recovered:

```sql
SQL> insert into test values (sysdate);

1 row created.

SQL> select * from test order by a desc;

A
-------------------
2012/03/19 20:58:18
2012/03/07 11:21:53
...
```

This change has not yet been archived so will be found in the online redo logs. The following is the RMAN recovery & restore process.

The first step is to take the affected tablespace(s) offline:

```sql
RMAN> sql 'alter tablespace users offline';
sql statement: alter tablespace users offline
```

The next step will restore the datafiles in this tablespace from a suitable dump. RMAN will choose the dump in order to reduce the amount of work it has to do to complete the restoration:

```sql
RMAN> restore tablespace users;
```
```text
Starting restore at 2012/03/19 21:01:25
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00006 to /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /media/oracle_backups/ant12/2nn3ict8_1_1
channel ORA_DISK_1: piece handle=/media/oracle_backups/ant12/2nn3ict8_1_1 tag=TAG20120217T165152
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 2012/03/19 21:01:26
```

With the dump of the datafile(s) restored, I next need to recover the tablespace to the current state:

```sql
RMAN> recover tablespace users;
```
```text
Starting recover at 2012/03/19 21:01:32
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 2012/03/19 21:01:33
```

Finally, bring the tablespace back online:

```sql
RMAN> sql 'alter tablespace users online';
sql statement: alter tablespace users online
```

And to be sure that I have indeed recovered the tablespace completely up to date, I re-ran the above query in my SQL\*Plus session again:

```sql
SQL> select * from test order by a desc;

A
-------------------
2012/03/19 20:58:18
2012/03/07 11:21:53
...
```

If the SYSTEM or UNDO tablespace need restoring and recovery then the database has to be `mounted`, as follows:

```sql
RMAN> shutdown

database closed
database dismounted
Oracle instance shut down
```

```sql
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

```sql
RMAN> restore tablespace system;
```
```text
Starting restore at 2012/03/19 21:14:34
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=17 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /srv/nffs/oradata/ant12/data/system01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_nnndf_TAG20120217T161210_7mwz2tyv_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_nnndf_TAG20120217T161210_7mwz2tyv_.bkp tag=TAG20120217T161210
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:25
Finished restore at 2012/03/19 21:15:00
```

```sql
RMAN> recover tablespace system;
```
```text
Starting recover at 2012/03/19 21:15:07
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 13 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_13_7mwzxtxg_.arc
archived log for thread 1 with sequence 14 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_14_7mwzxx1n_.arc
archived log for thread 1 with sequence 15 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_15_7mwzxxkb_.arc
archived log for thread 1 with sequence 16 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_16_7mwzxzvm_.arc
archived log for thread 1 with sequence 17 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_17_7mwzy01m_.arc
archived log for thread 1 with sequence 18 is already on disk as file /srv/nffs/flashback_area/ant12/NORM/archivelog/2012_03_19/o1_mf_1_18_7ph695lw_.arc
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_13_7mwzxtxg_.arc thread=1 sequence=13
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_14_7mwzxx1n_.arc thread=1 sequence=14
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_15_7mwzxxkb_.arc thread=1 sequence=15
media recovery complete, elapsed time: 00:00:02
Finished recover at 2012/03/19 21:15:10
```

```sql
RMAN> alter database open;
database opened
```

## Recover Datafiles

Recovering datafiles instead of a complete tablespace could save you a lot of downtime for the affected users. As ever, the affected datafile(s) need to be offline in order to be recovered. The remainder of the process is as simple as restoring and recovering a tablespace so the following demonstration of a datafile recovery needs little comment, however, as before, if SYSTEM or UNDO are affected, the database needs to be mounted, not open.

```sql
RMAN> sql 'alter database datafile 5 offline';
sql statement: alter database datafile 5 offline
```

```sql
RMAN> restore datafile 5;
```
```text
Starting restore at 2012/03/19 21:40:50
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00005 to /srv/nffs/oradata/ant12/data/tools01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_nnndf_TAG20120217T161511_7mwz8hvx_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_17/o1_mf_nnndf_TAG20120217T161511_7mwz8hvx_.bkp tag=TAG20120217T161511
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 2012/03/19 21:40:52
```
```sql
RMAN> recover datafile 5;
```
```text
Starting recover at 2012/03/19 21:40:57
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 13 is already on disk as file /srv/nffs/flashback_area/ant12/ANT12/archivelog/2012_02_17/o1_mf_1_13_7mwzxtxg_.arc
...
media recovery complete, elapsed time: 00:00:00
Finished recover at 2012/03/19 21:40:58
```
```sql
RMAN> sql 'alter database datafile 5 online';
sql statement: alter database datafile 5 online
```

Of course, the above assumes you know the datafile number, the `report schema` command helps here. You can always run the restore and recovery using the *full filename in single quotes*, but I find the data file number method to be easier and less prone to my abysmal typing skills!

If you have multiple files to restore and recover, separate them with a comma in the normal fashion:

```sql
RMAN> sql 'alter database datafile 5, 6 offline';
...
RMAN> restore datafile 5, 6;
...
RMAN> recover datafile 5, 6;
...
RMAN> sql 'alter database datafile 5, 6 online';
```

## Recover Individual Blocks

If you can save time by restoring and recovering just a datafile or two rather than a complete tablespace, then how about restoring and recovering a block or two, that's will save even more time surely?

RMAN can help you out here as well, and you don't need the datafile(s) to be offline at the time.

```sql
RMAN> recover datafile 5 block 24;
```
```text
Starting recover at 2012/03/19 21:49:47
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:00

Finished recover at 2012/03/19 21:49:47
```

In this case, even the SYSTEM and UNDO tablespaces can be recovered while online. You should note that there is no need to *restore* the data block first, just run the *recover* command.

See the RMAN manual for many other formats of this useful command.
