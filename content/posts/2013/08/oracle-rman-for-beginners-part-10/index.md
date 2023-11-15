---
title: "Oracle RMAN for Beginners – Part 10"
date: "2013-08-13"
categories: 
  - "oracle"
---

A slight variation on the incremental backups. In this (short) article, I demonstrate the use of database file copy backups which are themselves updated on a regular basis to avoid having to restore and recover using numerous incremental backups.

### What's Going On Here?

This took me a wee while to get my head around. We can take a backup of the database, in incremental form, and then, use our nightly incremental backups to _update the backup itself to the latest state of the database_. This means that although we are taking a regular incremental backup, a restore will require much less work as the backup is nearly up to date.

This requires the use of file copy backups, as opposed to backupsets, and was hinted at in [the previous article](/posts/2013/08/oracle-rman-for-beginners-part-9/ "Oracle RMAN for Beginners – Part 9") where I introduced incremental backups.

Image copies of the database are usually full copies. You would take a copy of the data files and do that every night, however, with RMAN, we can take an initial file copy of the database and then, incrementally, update that with each day's changes. It's not quite as simple as that, as we shall see, but bear with me!

### File Copy Updates

In order to perform this type of backup, we need to use the following commands in a run block:

```sql
RMAN> run {                                                   
2> recover copy of database with tag "Daily";
3> backup incremental level 1
4> for recover of copy
5> with tag "Daily"
6> database;
7> }
```

So, what happens?

- Day 1 - there is nothing to recover and no file copy backup of the database exists. The commands will therefore create a new file copy of the database. The autobackup of the controlfile etc creates a backup as normal. The `list backup summary` command will show the latter while the `list copy of database` command will show details of the former. Unfortunately, there is no `summary` option on the `list copy` command.

- Day 2 - Yesterday's file copy still cannot be recovered because there is, as yet, no incremental backups to apply. However, this pass through will create a new level 1 incremental backup. As ever, whatever has been configured to autobackup, will do so.

- Subsequent days - Yesterday's level 1 backup will be used to recover the file copy of the database taken on Day 1. A new level 1 incremental backup will be created afterwards and finally, the usual autobackup requirements will be carried out.

With the exception of the first two runs of these commands, there will be a file copy of the database as of the end of _yesterday's level 1 backup_ and a new level 1 backup from today. A recovery from this backup situation will involve restoring from the file copy and recovering from the level 1 backup - and any online-redo logs as required.

### A Worked Example

As ever, set NLS_DATE_FORMAT in the shell prior to running RMAN - this gives better quality date formatting in the output and listings.

```bash
$ export NLS_DATE_FORMAT='dd/mm/yyyy hh24:mi:ss'

$rman target /
```

In this demonstration, assume that this is the first time this set of commands has ever been run - or, at least, the first time for this tag value.

```sql
RMAN>### DAY ONE ###
2> run {                                                   
recover copy of database with tag "Daily Backup";
backup incremental level 1
for recover of copy
with tag "Daily Backup"
database;
}
```
```text
Starting recover at 13/08/2013 10:02:53
using channel ORA_DISK_1
no copy of datafile 1 found to recover
...
Finished recover at 13/08/2013 10:02:53

Starting backup at 13/08/2013 10:02:53
using channel ORA_DISK_1
no parent backup or copy of datafile 1 found
...

channel ORA_DISK_1: starting datafile copy
input datafile file number=00004 name=/srv/nffs/oradata/ant12/data/perfstat01_01.dbf
output file name=/srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_perfstat_90mxky4f_.dbf tag=DAILY BACKUP RECID=118 STAMP=823341801
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:35

...

Finished backup at 13/08/2013 10:04:53

Starting Control File and SPFILE Autobackup at 13/08/2013 10:04:53
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_13/o1_mf_s_823341893_90mxopfw_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 13/08/2013 10:04:56
```

```sql
RMAN> list backup summary;
```
```text
List of Backups
===============
Key     TY LV S Device Type Completion Time     #Pieces #Copies Compressed Tag
------- -- -- - ----------- ------------------- ------- ------- ---------- ---
...
117     B  F  A DISK        13/08/2013 10:04:55 1       1       NO         TAG20130813T100453

RMAN> list copy;
...

List of Datafile Copies
=======================

Key     File S Completion Time     Ckp SCN    Ckp Time           
------- ---- - ------------------- ---------- -------------------
119     1    A 13/08/2013 10:03:52 682467     13/08/2013 10:03:29
        Name: /srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_system_90mxm1cv_.dbf
        Tag: DAILY BACKUP

...
```
So, we can see that the `recover` command did nothing as there was no file copy to be recovered. The `backup` command started by noting that the requested level 1 incremental backup didn't have a parent (level zero) backup and so created a file copy backup with our requested tag.

When we look at the list of backups, we see only one - the tag matches the date and time of the controlfile autobackup and this is confirmed if we `list backupset 117`.

The following day, we execute exactly the same set of commands.
```sql
RMAN> ### DAY TWO ###
2>run {                                                   
recover copy of database with tag "Daily Backup";
backup incremental level 1
for recover of copy
with tag "Daily Backup"
database;
}
```
```text
Starting recover at 13/08/2013 10:05:53
using channel ORA_DISK_1
no copy of datafile 1 found to recover
...
Finished recover at 13/08/2013 10:05:53

Starting backup at 13/08/2013 10:05:53
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 1 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00004 name=/srv/nffs/oradata/ant12/data/perfstat01_01.dbf
...

channel ORA_DISK_1: starting piece 1 at 13/08/2013 10:05:53
channel ORA_DISK_1: finished piece 1 at 13/08/2013 10:05:54
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mxqkq9_.bkp tag=DAILY BACKUP comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 13/08/2013 10:05:54

Starting Control File and SPFILE Autobackup at 13/08/2013 10:05:54
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_13/o1_mf_s_823341954_90mxqm3y_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 13/08/2013 10:05:55
```
```sql
RMAN> list backup summary;
```
```text
List of Backups
===============
Key     TY LV S Device Type Completion Time     #Pieces #Copies Compressed Tag
------- -- -- - ----------- ------------------- ------- ------- ---------- ---
...
117     B  F  A DISK        13/08/2013 10:04:55 1       1       NO         TAG20130813T100453
118     B  1  A DISK        13/08/2013 10:05:54 1       1       NO         DAILY BACKUP
119     B  F  A DISK        13/08/2013 10:05:55 1       1       NO         TAG20130813T100554
```
```sql
RMAN> list copy;
```
```text
...

List of Datafile Copies
=======================

Key     File S Completion Time     Ckp SCN    Ckp Time           
------- ---- - ------------------- ---------- -------------------
119     1    A 13/08/2013 10:03:52 682467     13/08/2013 10:03:29
        Name: /srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_system_90mxm1cv_.dbf
        Tag: DAILY BACKUP
...
```

At the end of day two, we see that the file copy backup has still not been recovered. This is because there wasn't a level one incremental backup created yesterday. Remember, the recovery applies the latest incremental backup to the file copy and that incremental backup has not been created yet.

We then see that a new level one incremental backup is being created with our requested tag value, followed by the normal autobackup stuff.

Listing the backups shows that backups 118 and 119 have been created today. The latter is the autobackup and the former is our new (first) incremental backup. It is a coincidence that the file copy is also 119 - this need not always be the case, as tomorrow will show.

And the third, and subsequent, runs produce output similar to that shown below.
```sql
RMAN> ### DAY THREE AND SUBSEQUENT DAYS ###
2> run {                                                   
recover copy of database with tag "Daily Backup";
backup incremental level 1
for recover of copy
with tag "Daily Backup"
database;
}
```
```text
Starting recover at 13/08/2013 10:06:45
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile copies to recover
recovering datafile copy file number=00001 name=/srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_system_90mxm1cv_.dbf
...

channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mxqkq9_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mxqkq9_.bkp tag=DAILY BACKUP
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished recover at 13/08/2013 10:06:46

Starting backup at 13/08/2013 10:06:47
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 1 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00004 name=/srv/nffs/oradata/ant12/data/perfstat01_01.dbf
...

channel ORA_DISK_1: starting piece 1 at 13/08/2013 10:06:47
channel ORA_DISK_1: finished piece 1 at 13/08/2013 10:06:48
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mxs7ob_.bkp tag=DAILY BACKUP comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 13/08/2013 10:06:48

Starting Control File and SPFILE Autobackup at 13/08/2013 10:06:48
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_13/o1_mf_s_823342008_90mxs90z_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 13/08/2013 10:06:49
```
```sql
RMAN> list backup summary;
```
```text
List of Backups
===============
Key     TY LV S Device Type Completion Time     #Pieces #Copies Compressed Tag
------- -- -- - ----------- ------------------- ------- ------- ---------- ---
...
117     B  F  A DISK        13/08/2013 10:04:55 1       1       NO         TAG20130813T100453
118     B  1  A DISK        13/08/2013 10:05:54 1       1       NO         DAILY BACKUP
119     B  F  A DISK        13/08/2013 10:05:55 1       1       NO         TAG20130813T100554
120     B  1  A DISK        13/08/2013 10:06:47 1       1       NO         DAILY BACKUP
121     B  F  A DISK        13/08/2013 10:06:49 1       1       NO         TAG20130813T100648
```
```sql
RMAN> list copy;
```
```text
...

List of Datafile Copies
=======================

Key     File S Completion Time     Ckp SCN    Ckp Time           
------- ---- - ------------------- ---------- -------------------
131     1    A 13/08/2013 10:06:45 682600     13/08/2013 10:05:53
        Name: /srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_system_90mxm1cv_.dbf
        Tag: DAILY BACKUP
...
```

We see from the above that we have finally recovered the image copy of the database from yesterday's level one backup. So that takes the image copy of the data files up to date until the end of yesterday's backup. We also see that a brand new level one backup was taken with today's changes copied. This will be recovered into the file copy backup on tomorrow's run of these commands. The file copy of the database will always be as up to date as of _yesterday_ and not as of today.

Looking at the list of backups, we can see the most recent ones are the tagged daily backup which is the level one created just now and the usual autobackup one.

Looking at the file copies, we see that yesterday's key numbers have gone, and new ones are in place as are the checkpoint SCN numbers - the files are updated in other words.

### Running Recoveries

So, what happens when we need to recover a tablespace, for example?

```sql
RMAN> sql "alter tablespace users offline";
sql statement: alter tablespace users offline

RMAN> restore tablespace users;
```
```text
Starting restore at 13/08/2013 10:57:59
using channel ORA_DISK_1

channel ORA_DISK_1: restoring datafile 00006
input datafile copy RECID=140 STAMP=823343078 file name=/srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_users_90mxonmc_.dbf
destination for restore of datafile 00006: /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: copied datafile copy of datafile 00006
output file name=/srv/nffs/oradata/ant12/data/users01.dbf RECID=0 STAMP=0
Finished restore at 13/08/2013 10:58:00

RMAN> recover tablespace users;

Starting recover at 13/08/2013 10:58:06
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00006: /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mytqy3_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mytqy3_.bkp tag=DAILY BACKUP
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:00

starting media recovery
media recovery complete, elapsed time: 00:00:00

Finished recover at 13/08/2013 10:58:06
```
```sql
RMAN> sql "alter tablespace users online";
sql statement: alter tablespace users online
```

The first thing we see is that the file with recid = 140 is being restored as a file copy. We can see what this means with the following:

```sql
RMAN> list datafilecopy 140;
```
```text
List of Datafile Copies
=======================

Key     File S Completion Time     Ckp SCN    Ckp Time           
------- ---- - ------------------- ---------- -------------------
140     6    A 13/08/2013 10:24:38 682690     13/08/2013 10:06:47
        Name: /srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_users_90mxonmc_.dbf
        Tag: DAILY BACKUP
```

Or, we could have found out what files would have been restored for the tablespace as follows:

```sql
RMAN> list copy of tablespace users;
```
```text
List of Datafile Copies
=======================

Key     File S Completion Time     Ckp SCN    Ckp Time           
------- ---- - ------------------- ---------- -------------------
140     6    A 13/08/2013 10:24:38 682690     13/08/2013 10:06:47
        Name: /srv/nffs/flashback_area/ant12/ANT12/datafile/o1_mf_users_90mxonmc_.dbf
        Tag: DAILY BACKUP
```

Either way, we now know that a file copy was restored. The recovery phase isn't so easy to decode, but we can see which backuppiece was used. It is highlighted in bold above. Checking what that actually is gives the following:

```sql
RMAN> list backuppiece "/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mytqy3_.bkp";
```
```text
List of Backup Pieces
BP Key  BS Key  Pc# Cp# Status      Device Type Piece Name
------- ------- --- --- ----------- ----------- ----------
122     122     1   1   AVAILABLE   DISK        /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mytqy3_.bkp
```

This gives the backupset key number, which also happens to be 122. We can interrogate this next:

```sql
RMAN> list backupset 122;
```
```text
List of Backup Sets
===================

BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
122     Incr 1  464.00K    DISK        00:00:01     13/08/2013 10:24:40
        BP Key: 122   Status: AVAILABLE  Compressed: NO  Tag: DAILY BACKUP
        Piece Name: /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_13/o1_mf_nnnd1_DAILY_BACKUP_90mytqy3_.bkp
  List of Datafiles in backup set 122
  File LV Type Ckp SCN    Ckp Time            Name
  ---- -- ---- ---------- ------------------- ----
  1    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/system01.dbf
  2    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/sysaux01.dbf
  3    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/undotbs01.dbf
  4    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/perfstat01_01.dbf
  5    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/tools01.dbf
  6    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/users01.dbf
  7    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/audit01_01.dbf
  8    1  Incr 683288     13/08/2013 10:24:39 /srv/nffs/oradata/ant12/data/utility01_01.dbf
```
The output from the above shows that we are applying a level 1 incremental backup. Phew! At least we now know. There must be an easier way that this to find out though, surely?

Restoring individual data files, the whole database, or just a block or two is as easy as shown in previous examples. Don't forget, tablespaces and data files need to be offline before they can be restored and if the tablespace or the datafiles are SYSTEM or UNDO, then the whole database needs to be in mount mode.
