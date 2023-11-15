---
title: "Oracle RMAN for Beginners – Part 9"
date: "2013-08-12"
categories: 
  - "oracle"
---

It's been a while since [the previous post in this series](/posts/2012/05/oracle-rman-for-beginners-part-8/ "Oracle RMAN for Beginners – Part 8"), but I'm back again. This time out, we are looking at incremental backups. What they are, how they work, and how - of course - to take them and use the to restore and recover your databases.

### More Terminology

What exactly is an incremental backup? Previously, this series has shown you how to take a _full_ backup be that of the database, archived logs, tablespaces and data files. Each time, for example, you back up a full database, you copy everything - even data that have not changed. This does make it simpler to restore and recover the database as you only require the most recent backup, but it can take quite some time if you are backing up a multi-terabyte database every night.

An incremental backup works on the principle that if you already have a backup, and only some of your data has changed, then surely it will be quicker to just backup the changed data? This is exactly how it works.

### Incremental Backup Levels

There are two incremental backup levels:

- Level 0 (zero) - this is almsot the equivalent of a full backup, but it prepares the way for any following incremental backups, of any kind (see below), by giving you a starting position from which to work.
    
    > You must start from a level zero incremental backup and not a previous full backup. Although both backup everything, only the level zero incremental one can be used as a parent backup for future level one backups.
    
- Level 1 (one) - this is your normal incremental backup level. It only backs up blocks which changed recently. Which blocks it backs up are dependent on the backup type - which is described below. A level 1 incremental backup should be smaller and faster than a full or level 0 backup as it _normally_ wouldn't copy the entire database. However, the final size does depend on the number of changes, and as mentioned, the backup type.

### Incremental Backup Types

There are two different incremental backup types:

- Differential - the default incremental backup type. This backs up only those blocks which have been changed since the previous _level 0_ or _level 1_ incremental backups.
- Cumulative - this incremental backup will backup all changed blocks since the previous _level 0_ incremental backup.

All incremental backups use backup sets, not file copies. However, see the next article in the series for details of something which may, at first glance, appear to contradict this statement! (Merged file copy incremental backups.)

### A Contrived Example

The following backup strategy should hopefully clarify things:

- On Monday, you take a _level 0_ incremental backup.
- On Tuesday, you take a _differential level 1_ incremental backup. This means that only those blocks which changed since the previous level 0 or level 1 backup will be copied. This means, changes since _Monday's level 0_ backup.
- On Wednesday, you take another _differential level 1_ incremental backup. This means that only those blocks which changed since the previous level 0 or level 1 backup will be copied. This means, changes since _Tuesday's level 1_ backup.
- On Thursday, you take a _cumulative level 1_ incremental backup. This backup will copy all changed blocks from the previous level 0 backup - which took place on _Monday_.
- On Friday, you take a _differential level 1_ incremental backup. This backup will copy all changed blocks from the previous level 0 or level 1 backup - which took place on _Thursday_ - because you can mix and match the level 1 backup types without any worries.
- On Saturday, you take another _differential level 1_ incremental backup. This backup will copy all changed blocks from the previous level 0 or level 1 backup - which took place on _Friday_.
- You have Sunday off to be with your family!

Why would you want to take a cumulative backup at certain points when you are already taking differential backups? Well, even though you are taking incremental backups, it is possible that there will be a lot of changes. Cumulative backups help with the restoration of the database after a failure - they can reduce the amount of work you need to do, tapes to be located etc.

Here's what RMAN would do in the event of a database failure on any of the days when the failure is _prior_ to the backups being successfully completed. Assume that the entire database will require restoring and recovery.

- Tuesday - Restore Monday's backup and use the archived logs to recover the database.
- Wednesday - As for Tuesday, but Tuesday's level 1 backup would be required in addition to the archived logs for recovery.
- Thursday - As for Wednesday, but Wednesday's level 1 backup would be required in addition to the archived logs for recovery.
- Friday - Restore Monday's backup, recover using Thursday's cumulative backup, in addition to the archived logs needed to bring us back up to date.
- Saturday - As per Friday, but also requires Friday's backup as part of the recovery.
- Sunday - As per Saturday but add in Saturday's backup as part of the recovery.
- Monday - As per Sunday, but you better make sure that you have all the archived logs that were created after the Saturday backup in order that you can recover bang up to date! Maybe you should consider reviewing the above backup strategy? ;-)

So you can see, using cumulative backups reduces the number of incremental backups you need to use to recover the database. RMAN will choose the best backup it knows about - in the controlfile or the catalogue - to _restore_ the database - this may be a level 0 incremental backup.

When you _recover_ the database, RMAN will choose to use archived logs or an incremental level 1 backup or a mixture of both, as required, to get the database back to where you requested it.

Incremental backups take less time to produce, but can require a little more thought when trying to restore and recover a database (tablespace etc) as you may need to keep more backups in the backup location to allow an adequate recovery window.

Full backups are easier, RMAN will most likely restore the most recent backup and use the archived logs to recover the database, but as mentioned, these backups may be very much larger.

### Slightly Technical Stuff

Under normal conditions, RMAN will perform an incremental backup by reading each and every (non-virgin) block into the buffer and checking if the SCN is higher (or the same according to _Kuhn, Alapati and Nanda_) than the SCN of the most recent level 0 (or level 1 depending on the incremental backup type) backup. If the block's SCN is higher, this block requires backup.

You can make this process a lot quicker, especially with large databases, by turning on _Block Change Tracking_. This process uses a binary file to record the blocks that have changed over the normal day to day running of the database, since the most recent level 0/level 1 backup.

If this feature is turned on, when RMAN performs an incremental backup at level 1, it doesn't need to read each and every (non-virgin) block into the buffer as it knows already which blocks need copying. You can turn on block change tracking as follows - assuming that we are not using Oracle Managed files:

SQL> alter database enable block change tracking using file '/path/to/file/change_tracking.chg' reuse;
Database altered.

The filename will be created if it doesn't exist, but the path should already exists or the command will fail. If the file already exists, it will be overwritten. This could cause the loss of existing change tracking data, so check first, to see if it is already enabled:

```sql
SQL> select * from v$block_change_tracking;
```
```text
STATUS     FILENAME                                                            BYTES
---------- ------------------------------------------------------------------ -----------
ENABLED    /srv/nffs/flashback_area/ant12/change_tracking/change_tracking.chg  11599872
```

Block change tracking can be turned off, if desired, as follows:

```sql
SQL> alter database disable block_change_tracking;
Database altered.
```

You cannot specify a size for the change tracking file. Oracle will allocate the required space itself, based on the database size, the number of data files in the database and the number of redo threads that are enabled. In 11gR2, the file is initially created at 10Mb and grows in 10Mb chunks as new space is required. There is 320 Kb allocated in the file for each data file in the database. (_Information from Kuhn, Alapati & Nanda._).

### Enough Talking, Lets Do Backups!

As ever, we should consider setting `NLS_DATE_FORMAT` in the shell before we begin. This way, we get better date's when we look at the dates and times of our backups. You cannot set this within RMAN itself, sadly.

```bash
$ export NLS_DATE_FORMAT='dd/mm/yyyy hh24:mi:ss'

$ rman target /
```

Once we are in RMAN and connected to the target database (and catalogue, if one is in use) we can start the incremental process by taking an initial level 0 incremental backup. You will note from the following that I am including the archived logs too. You can't incrementally back those up, so they get copied in the normal manner.

```sql
RMAN> backup incremental level 0 database tag "DB Level 0";
```
```text
Starting backup at 12/08/2013 15:51:59
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 0 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00004 name=/srv/nffs/oradata/ant12/data/perfstat01_01.dbf
input datafile file number=00001 name=/srv/nffs/oradata/ant12/data/system01.dbf
input datafile file number=00002 name=/srv/nffs/oradata/ant12/data/sysaux01.dbf
input datafile file number=00007 name=/srv/nffs/oradata/ant12/data/audit01_01.dbf
input datafile file number=00008 name=/srv/nffs/oradata/ant12/data/utility01_01.dbf
input datafile file number=00003 name=/srv/nffs/oradata/ant12/data/undotbs01.dbf
input datafile file number=00005 name=/srv/nffs/oradata/ant12/data/tools01.dbf
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting piece 1 at 12/08/2013 15:52:00
channel ORA_DISK_1: finished piece 1 at 12/08/2013 15:52:15
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp tag=DB LEVEL 0 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:15
Finished backup at 12/08/2013 15:52:15

Starting Control File and SPFILE Autobackup at 12/08/2013 15:52:15
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_12/o1_mf_s_823276335_90kxnzkh_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 12/08/2013 15:52:16

Don't forget the archived logs though! Even though we are doing an incremental backup, the archived logs are still required.
```
```sql
RMAN>  backup incremental level 0 archivelog all delete input tag "ARC Level 0";
```
```text
Starting backup at 12/08/2013 15:54:14
current log archived
using channel ORA_DISK_1
channel ORA_DISK_1: starting archived log backup set
channel ORA_DISK_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=57 RECID=57 STAMP=823275596
input archived log thread=1 sequence=58 RECID=58 STAMP=823275613
input archived log thread=1 sequence=59 RECID=59 STAMP=823276422
input archived log thread=1 sequence=60 RECID=60 STAMP=823276454
channel ORA_DISK_1: starting piece 1 at 12/08/2013 15:54:14
channel ORA_DISK_1: finished piece 1 at 12/08/2013 15:54:15
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_annnn_ARC_LEVEL_0_90kxrpvq_.bkp tag=ARC LEVEL 0 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
channel ORA_DISK_1: deleting archived log(s)
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2013_08_12/o1_mf_1_57_90kwxw9z_.arc RECID=57 STAMP=823275596
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2013_08_12/o1_mf_1_58_90kwyfhj_.arc RECID=58 STAMP=823275613
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2013_08_12/o1_mf_1_59_90kxqpck_.arc RECID=59 STAMP=823276422
RMAN-08138: WARNING: archived log not deleted - must create more backups
archived log file name=/srv/nffs/flashback_area/ant12/ANT12/archivelog/2013_08_12/o1_mf_1_60_90kxrpl4_.arc thread=1 sequence=60
Finished backup at 12/08/2013 15:54:15

Starting Control File and SPFILE Autobackup at 12/08/2013 15:54:15
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_12/o1_mf_s_823276456_90kxrr72_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 12/08/2013 15:54:17
```

Yes, I know, it looks silly! How on earth can you incrementally backup an archived log? Well, the whole log file will be copied because each and every block has changed! But in effect, it's a normal archived log backup.

> Now, you may be wondering why I didn't just use the following command:
> 
> ```sql
> RMAN> backup incremental level 0 database 
> 2> plus archivelog all delete input
> 3> tag "DB Level 0";
> ```
> 
> The command _does_ work, and _does_ backup both the database and the archived logs, however, the archived logs will be copied first and will take the _requested tag_ - "DB Level 0", then the archived logs on disc will be deleted as normal - according to the deletion policy configuration. When the database is backed up, the tag _is not used_, so the actual database backup gets an RMAN generated tag and not the one we wanted. The current redo log is then archived and backed up - _getting the requested tag again_ - and finally the controlfile and spfile are backed up according to the autobackup policy and again, get an RMAN generated tag.
> 
> It appears that whatever tag is requested is only applied to the first backupset in the backup. If you wish the database and archived log backups to be tagged in this way, then run separate backups with an appropriate tag. The autobackup of the controlfile and/or spfile will not use the requested tag.

That's the initial level 0 backup. We can now do some work in the database - I shall not show that here, but you can be assured that I have made some changes to tables in the working schema I'm using - and then backup only the changes.

```sql
RMAN> backup incremental level 1 database tag "DB LEvel 1";
```
```text
Starting backup at 12/08/2013 16:07:18
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 1 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00004 name=/srv/nffs/oradata/ant12/data/perfstat01_01.dbf
input datafile file number=00001 name=/srv/nffs/oradata/ant12/data/system01.dbf
input datafile file number=00002 name=/srv/nffs/oradata/ant12/data/sysaux01.dbf
input datafile file number=00007 name=/srv/nffs/oradata/ant12/data/audit01_01.dbf
input datafile file number=00008 name=/srv/nffs/oradata/ant12/data/utility01_01.dbf
input datafile file number=00003 name=/srv/nffs/oradata/ant12/data/undotbs01.dbf
input datafile file number=00005 name=/srv/nffs/oradata/ant12/data/tools01.dbf
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting piece 1 at 12/08/2013 16:07:18
channel ORA_DISK_1: finished piece 1 at 12/08/2013 16:07:19
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1_90kyk6qw_.bkp tag=DB LEVEL 1 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 12/08/2013 16:07:19

Starting Control File and SPFILE Autobackup at 12/08/2013 16:07:19
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_12/o1_mf_s_823277239_90kyk81j_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 12/08/2013 16:07:20
```

That's it done, it took all of 1 second to backup the changes I made. Because I didn't specify to perform a cumulative backup, RMAN defaulted to differential, as described above. If I now carry out some more work, and take another level 1 differential backup, I'll get only the changed blocks since the backup we just carried out.

```sql
RMAN>  backup incremental level 1 database tag "DB Level 1 - part 2";
```

I have omitted the output from this one as it's almost identical to the one above. Also, you will note, I am not backing up the archived logs - you can assume that I am, because I should be, I'm not showing the output though.

And now, we shall take a cumulative incremental backup, which will create a new backup consisting of all the blocks that changed since the previous level zero backup. This is effectively, everything in the two differential backups taken above.

```sql
RMAN>  backup incremental level 1 cumulative database tag "DB Level 1 - cumulative";
```
```text
Starting backup at 12/08/2013 16:12:21
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 1 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00004 name=/srv/nffs/oradata/ant12/data/perfstat01_01.dbf
...
channel ORA_DISK_1: starting piece 1 at 12/08/2013 16:12:21
channel ORA_DISK_1: finished piece 1 at 12/08/2013 16:12:22
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp tag=DB LEVEL 1 - CUMULATIVE comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 12/08/2013 16:12:22

Starting Control File and SPFILE Autobackup at 12/08/2013 16:12:22
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_12/o1_mf_s_823277542_90kytq64_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 12/08/2013 16:12:23
```

You will note that there is little indication that this is a cumulative backup, other than the tag I have used to show me what it is! How can we discover what level a backup is, what type and so on?

### Listing Backups

We can see the backups, and their tags, as follows:

```sql
RMAN> list backup summary;
```
```text
List of Backups
===============
Key     TY LV S Device Type Completion Time     #Pieces #Copies Compressed Tag
------- -- -- - ----------- ------------------- ------- ------- ---------- ---
62      B  A  A DISK        12/08/2013 15:38:01 1       1       NO         FULL BACKUP
63      B  F  A DISK        12/08/2013 15:38:10 1       1       NO         TAG20130812T153802
64      B  A  A DISK        12/08/2013 15:38:18 1       1       NO         FULL BACKUP
65      B  F  A DISK        12/08/2013 15:38:20 1       1       NO         TAG20130812T153819
70      B  0  A DISK        12/08/2013 15:52:07 1       1       NO         DB LEVEL 0
71      B  F  A DISK        12/08/2013 15:52:16 1       1       NO         TAG20130812T155215
72      B  A  A DISK        12/08/2013 15:53:42 1       1       NO         ARC LEVEL 0
73      B  F  A DISK        12/08/2013 15:53:44 1       1       NO         TAG20130812T155343
74      B  A  A DISK        12/08/2013 15:54:14 1       1       NO         ARC LEVEL 0
75      B  F  A DISK        12/08/2013 15:54:16 1       1       NO         TAG20130812T155416
76      B  1  A DISK        12/08/2013 16:07:19 1       1       NO         DB LEVEL 1
77      B  F  A DISK        12/08/2013 16:07:20 1       1       NO         TAG20130812T160719
78      B  1  A DISK        12/08/2013 16:09:29 1       1       NO         DB LEVEL 1 - PART 2
79      B  F  A DISK        12/08/2013 16:09:31 1       1       NO         TAG20130812T160931
80      B  1  A DISK        12/08/2013 16:12:21 1       1       NO         DB LEVEL 1 - CUMULATIVE
81      B  F  A DISK        12/08/2013 16:12:23 1       1       NO         TAG20130812T161222
82      B  A  A DISK        12/08/2013 16:13:31 1       1       NO         ARC LEVEL 1
83      B  F  A DISK        12/08/2013 16:13:33 1       1       NO         TAG20130812T161332
```

In the above, KEY is the backupset key, TY is the backup type which in these examples is all _Backupset_ types, LV is the backup level - 0 (zero) is incremental level 0, 1 (one) is incremental level 1, A is archived logs and F is a full backup (or autobackup of spfile and/or controlfile). The S column is the status where A indicates an available backup, and finally, the TAG column displays our requested tags.

You can see from the various columns how the backups do appear to match up to our tags. With the exception of the RMAN generated tags for the autobackups of course. You will also see that regardless of how the tag was originally specified in the `backup` command, it is converted to upper case.

### Backing Up Parts of the Database

You do not have to carry out an incremental backup of the entire database. As with full backups, you can backup at the tablespace or data file level. You cannot backup individual data blocks though!

A tablespace backup would be as follows:

```sql
RMAN> backup incremental level 1
2> tablespace users
3> tag "TS USERS - Level 1";
```
```text
Starting backup at 12/08/2013 16:34:59
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 1 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting piece 1 at 12/08/2013 16:34:59
channel ORA_DISK_1: finished piece 1 at 12/08/2013 16:35:00
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TS_USERS___LEVEL_1_90l053hb_.bkp tag=TS USERS - LEVEL 1 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 12/08/2013 16:35:00

Starting Control File and SPFILE Autobackup at 12/08/2013 16:35:00
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_12/o1_mf_s_823278900_90l054z0_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 12/08/2013 16:35:01
```

While a couple of data files (in different tablespaces, as it happens) would be thus:

```sql
RMAN> backup incremental level 1
2> datafile 5,7;
```
```text
Starting backup at 12/08/2013 16:36:23
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental level 1 datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00007 name=/srv/nffs/oradata/ant12/data/audit01_01.dbf
input datafile file number=00005 name=/srv/nffs/oradata/ant12/data/tools01.dbf
channel ORA_DISK_1: starting piece 1 at 12/08/2013 16:36:23
channel ORA_DISK_1: finished piece 1 at 12/08/2013 16:36:24
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TAG20130812T163623_90l07qc1_.bkp tag=TAG20130812T163623 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 12/08/2013 16:36:24

Starting Control File and SPFILE Autobackup at 12/08/2013 16:36:24
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2013_08_12/o1_mf_s_823278984_90l07rml_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 12/08/2013 16:36:25
```

### Restoring Incremental Backups

You can, as mentions previously, restore from incremental backups. Given the choice, RMAN will choose the most appropriate backup to restore from, and when you recover, then either a level 1 incremental or the archived logs will be used to recover back to the most recent state of the database. Assuming you didn't specify an SCN or an until time etc of course!

Equally, as with full backups, you can restore to the database, tablespace, datafile or block level, as well as restoring the archived logs.

The following examples show each of these recoveries.

### Database Recovery

To restore the entire database, it must be shut down to the mounted state. This is no different from carrying out a restore from a full backup.

```sql
RMAN> shutdown
```
```text
database closed
database dismounted
Oracle instance shut down
```
```sql
RMAN> startup mount
```
```text
connected to target database (not started)
Oracle instance started
database mounted

Total System Global Area     768331776 bytes
...
```
```sql
RMAN> restore database;
```
```text
Starting restore at 12/08/2013 16:42:48
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /srv/nffs/oradata/ant12/data/system01.dbf
...

channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp
...

channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:01:16
Finished restore at 12/08/2013 16:44:04
```
```sql
RMAN> recover database;
```
```text
Starting recover at 12/08/2013 16:46:45
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /srv/nffs/oradata/ant12/data/system01.dbf
...

channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp
...

channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01

channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00006: /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TS_USERS___LEVEL_1_90l053hb_.bkp
...

channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:00

channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00005: /srv/nffs/oradata/ant12/data/tools01.dbf
destination for restore of datafile 00007: /srv/nffs/oradata/ant12/data/audit01_01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TAG20130812T163623_90l07qc1_.bkp
...

channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:00

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 12/08/2013 16:46:48
```
```sql
RMAN> alter database open;
database opened
```

So, RMAN chose to use the Incremental level zero backup to restore everything, then in the recovery phase, applied the cumulative level one backup as opposed to two separate differential level one backups, followed by the tablespace backup of the users tablespace and the data file backup of the data files we backed up above, rather than using archived log backups.

### Tablespace Recovery

To recover one or more tablespaces, the database can be open unless either (or both) the SYSTEM or UNDO tablespaces needs to be restored and recovered. In that case, the database must be mounted only.

The first example restores a single tablespace, USERS, and this can be done with the database open, but the tablespace must be taken off line. Users will only be affected if their work requires access to the tablespace(s) being restored.

```sql
RMAN> sql "alter tablespace users offline";
sql statement: alter tablespace users offline

RMAN> restore tablespace users;
```
```text
Starting restore at 12/08/2013 20:03:12
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00006 to /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp tag=DB LEVEL 0
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 12/08/2013 20:03:14
```
```sql
RMAN> recover tablespace users;
```
```text
Starting recover at 12/08/2013 20:03:20
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00006: /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp tag=DB LEVEL 1 - CUMULATIVE
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00006: /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TS_USERS___LEVEL_1_90l053hb_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TS_USERS___LEVEL_1_90l053hb_.bkp tag=TS USERS - LEVEL 1
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:00

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 12/08/2013 20:03:22
```
```sql
RMAN> sql "alter tablespace users online";
sql statement: alter tablespace users online
```

You can see, once again, that RMAN chose to restore the initial level zero incremental backup of the database, but only the USERS tablespace was restored from it. The recovery phase used the cumulative level one incremental backup plus the USERS tablespace backup that we took earlier. No archived logs were used in this recovery either.

The next example, restores the SYSTEM tablespace and as such, will require the database to be mounted.

```sql
RMAN> shutdown;
```
```text
...
Oracle instance shut down
```
```sql
RMAN> startup mount;
```
```text
...
database mounted

Total System Global Area     768331776 bytes
...
```
```sql
RMAN> restore tablespace system;
```
```text
Starting restore at 12/08/2013 20:08:15
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /srv/nffs/oradata/ant12/data/system01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp tag=DB LEVEL 0
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:26
Finished restore at 12/08/2013 20:08:41
```
```sql
RMAN> recover tablespace system;
```
```text
Starting recover at 12/08/2013 20:11:03
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /srv/nffs/oradata/ant12/data/system01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_backup incremental level 0 database tag "DB Level 0";area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp tag=DB LEVEL 1 - CUMULATIVE
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 12/08/2013 20:11:05
```
```sql
RMAN> alter database open;
database opened
```

This time RMAN restored from the same level zero backup as previously and recovered from the cumulative level one backup. Again, neither of the differential backups were used simply because the cumulative backup was the best choice for the recovery.

### Data File Recovery

This example shows the restoration and recovery of a data file that is not a member of the SYSTEM or UNDO tablespaces. For this, the data file(s) need to be taken offline, but the database can remain open. Users will only be affected if their work requires access to the data files being restored.

```sql
RMAN> sql "alter database datafile 5 offline";
sql statement: alter database datafile 5 offline

RMAN> restore datafile 5;
```
```text
Starting restore at 12/08/2013 20:16:39
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00005 to /srv/nffs/oradata/ant12/data/tools01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd0_DB_LEVEL_0_90kxnj9z_.bkp tag=DB LEVEL 0
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 12/08/2013 20:16:41
```
```sql
RMAN> recover datafile 5;
```
```text
Starting recover at 12/08/2013 20:16:47
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00005: /srv/nffs/oradata/ant12/data/tools01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_DB_LEVEL_1___CUMULAT_90kytoof_.bkp tag=DB LEVEL 1 - CUMULATIVE
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00005: /srv/nffs/oradata/ant12/data/tools01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TAG20130812T163623_90l07qc1_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2013_08_12/o1_mf_nnnd1_TAG20130812T163623_90l07qc1_.bkp tag=TAG20130812T163623
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:00

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 12/08/2013 20:16:49
```
```sql
RMAN> sql "alter database datafile 5 online";
sql statement: alter database datafile 5 online
```

You can see, as above, the backups used in restoration and recovery have been chosen by RMAN in order to most efficiently get the data file in question, back up to date.

Had we required to restore and recover data file(s) which are part of the SYSTEM or UNDO tablespaces, we would, once again, have had to have the database in mounted mode as opposed to open. This would, of course, affect all users of this particular database.

As this is very similar to the tablespace example given above for SYSTEM, I shall omit the example here.

### Data Block Recovery

RMAN incremental backups allow the recovery of a single, or multiple, data blocks. Why recover a whole data file when you can make matter much quicker by recovering only the affected blocks - assuming you know which ones are affected of course!

You can do this with the database open, even if the blocks are in the SYSTEM or UNDO tablespaces. You will notice below, that there is no need to restore the data blocks in question, you cannot! You simply recover them.

This first example recovers a data block in a normal tablespace:

```sql
RMAN> recover datafile 7 block 17;
```
```text
Starting recover at 12/08/2013 20:22:23
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 12/08/2013 20:22:25
```

And now, one block in the SYSTEM tablespace:

```sql
RMAN> recover datafile 1 block 9;
```
```text
Starting recover at 12/08/2013 20:25:13
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:00

Finished recover at 12/08/2013 20:25:13
```