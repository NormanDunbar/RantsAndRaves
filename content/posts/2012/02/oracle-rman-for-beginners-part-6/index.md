---
title: "Oracle RMAN for Beginners – Part 6"
date: "2012-02-17T00:00:09Z"
categories: 
  - "oracle"
---

In the past few instalments, I've looked mainly at databases running in NOARCHIVELOG mode. It's probably not the best way to run a production system but it is valid to do so.

For the safest systems and the ability to recover from numerous disasters without losing any data, the database should be run in ARCHIVELOG mode.

The remainder of the mini-series concentrates on the things you can do with RMAN when the database is running in ARCHIVELOG mode.

## Hot Backups

There's nothing to taking a hot backup that I haven't already shown you. Taking a backup of the full database in hot (or online) mode is exactly the same as taking a cold (offline) backup.

One difference is that you must backup the archived logs in addition to the database.

Another difference is that you can take a hot backup while the database is open, although you can also take one while the database is mounted.

In the following examples, I have set my archivelog deletion policy as follows:

```sql
RMAN> configure archivelog deletion policy backed up 2 times to disk;

new RMAN configuration parameters:
CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 2 TIMES TO DISK;
new RMAN configuration parameters are successfully stored
```

### Backup the Database and Archived Logs

To backup the database, including controlfile and spfile if autobackup is configured - and the archived logs, all I need to do is as follows:

```sql
RMAN> run {
2> backup full database;
3> backup archivelog all delete input;
4> }
```

An extract of the results of running the above is this:

```text
Starting backup at 2012/02/17 15:07:46
allocated channel: ORA\_DISK\_1
channel ORA\_DISK\_1: SID=45 device type=DISK
channel ORA\_DISK\_1: starting full datafile backup set
channel ORA\_DISK\_1: specifying datafile(s) in backup set
input datafile file number=00010 name=/srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01\_01.dbf
...
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 15:07:47
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 15:08:32
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_nnndf\_TAG20120217T150747\_7mwvb3sy\_.bkp tag=TAG20120217T150747 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:45
Finished backup at 2012/02/17 15:08:32

Starting backup at 2012/02/17 15:08:34
current log archived
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting archived log backup set
channel ORA\_DISK\_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=1 RECID=57 STAMP=775494253
input archived log thread=1 sequence=2 RECID=58 STAMP=775494254
input archived log thread=1 sequence=3 RECID=59 STAMP=775494255
input archived log thread=1 sequence=4 RECID=60 STAMP=775494258
input archived log thread=1 sequence=5 RECID=61 STAMP=775494274
input archived log thread=1 sequence=6 RECID=62 STAMP=775494514
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 15:08:35
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 15:08:38
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_annnn\_TAG20120217T150834\_7mwvcm3s\_.bkp tag=TAG20120217T150834 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:03
```

The archived logs are backed up to the FRA, as usual, and then deleted from their normal position in the FRA.This helps keep the FRA tidy of archived logs which are no longer necessary as enough backups have been taken.

Setting the archived log deletion policy sensibly means that you should never be without the logs you need to recover a restored database, either in full or up to a specific point in time, scn or log sequence number.

You may note, if you looked closely, that one of the first things RMAN did as part of the archived log backup, was to archive the current online redo log. This means that you will have all the archived logs necessary since the start of the backup (of the database) available in the event of a recovery being required.

As this is the first time my archived logs have been backed up, the attempt to delete them has not been successful, as the following extract shows:

```text
channel ORA\_DISK\_1: deleting archived log(s)
RMAN-08138: WARNING: archived log not deleted - must create more backups
archived log file name=/srv/nffs/flashback\_area/ant12/ANT12/archivelog/2012\_02\_17/o1\_mf\_1\_1\_7mwv3ddf\_.arc thread=1 sequence=1
...
RMAN-08138: WARNING: archived log not deleted - must create more backups
archived log file name=/srv/nffs/flashback\_area/ant12/ANT12/archivelog/2012\_02\_17/o1\_mf\_1\_6\_7mwvclf7\_.arc thread=1 sequence=6
Finished backup at 2012/02/17 15:08:38
```

These warnings _do not_ affect the backup. The archived logs in the FRA _have_ been successfully backed up (to the FRA) but because the deletion policy states that they can only be deleted, from the FRA, when they have already been backed up twice to disc, then they must remain on disc in the FRA.

Finally, the controlfile and spfile are automatically backed up:

```text
Starting Control File and SPFILE Autobackup at 2012/02/17 15:08:38
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775494518\_7mwvcpqn\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 15:08:41
```


It may be simpler to run the following command instead:

```sql
RMAN> backup full database plus archivelog delete input;
```

Looking at the log from the above command shows that the following steps take place:

- The current redo log is archived.
- The archived logs are (all) backed up.
- The deletion policy for archived logs is applied.
- The database is backed up.
- The (new) current redo log is archived.
- The newly archived log is backed up on its own.
- The controlfile and spfile are backed up - if configured to do so.

An extract from the log, showing the above steps, is as follows:

```text
Starting backup at 2012/02/17 15:37:19
current log archived
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting archived log backup set
channel ORA\_DISK\_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=1 RECID=57 STAMP=775494253
...
input archived log thread=1 sequence=7 RECID=63 STAMP=775496239
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 15:37:20
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 15:37:23
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_annnn\_TAG20120217T153720\_7mwx1j93\_.bkp tag=TAG20120217T153720 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:03

channel ORA\_DISK\_1: deleting archived log(s)
archived log file name=/srv/nffs/flashback\_area/ant12/ANT12/archivelog/2012\_02\_17/o1\_mf\_1\_1\_7mwv3ddf\_.arc RECID=57 STAMP=775494253
...
archived log file name=/srv/nffs/flashback\_area/ant12/ANT12/archivelog/2012\_02\_17/o1\_mf\_1\_6\_7mwvclf7\_.arc RECID=62 STAMP=775494514
RMAN-08138: WARNING: archived log not deleted - must create more backups
archived log file name=/srv/nffs/flashback\_area/ant12/ANT12/archivelog/2012\_02\_17/o1\_mf\_1\_7\_7mwx1hj5\_.arc thread=1 sequence=7
Finished backup at 2012/02/17 15:37:23

Starting backup at 2012/02/17 15:37:23
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting full datafile backup set
channel ORA\_DISK\_1: specifying datafile(s) in backup set
input datafile file number=00010 name=/srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01\_01.dbf
...
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 15:37:24
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 15:38:09
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_nnndf\_TAG20120217T153723\_7mwx1n4f\_.bkp tag=TAG20120217T153723 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:45
Finished backup at 2012/02/17 15:38:09

Starting backup at 2012/02/17 15:38:09
current log archived
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting archived log backup set
channel ORA\_DISK\_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=8 RECID=64 STAMP=775496289
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 15:38:09
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 15:38:10
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_annnn\_TAG20120217T153809\_7mwx31qd\_.bkp tag=TAG20120217T153809 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:01

channel ORA\_DISK\_1: deleting archived log(s)
RMAN-08138: WARNING: archived log not deleted - must create more backups
archived log file name=/srv/nffs/flashback\_area/ant12/ANT12/archivelog/2012\_02\_17/o1\_mf\_1\_8\_7mwx31h2\_.arc thread=1 sequence=8
Finished backup at 2012/02/17 15:38:10

Starting Control File and SPFILE Autobackup at 2012/02/17 15:38:10
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775496290\_7mwx33bn\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 15:38:13
```

### Backup Selected Tablespaces

One of the joys of running a database in ARCHIVELOG mode is the fact that it is possible to backup and recover individual tablespaces.

```sql
RMAN> backup tablespace users;
```
```text
Starting backup at 2012/02/17 15:59:07
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting full datafile backup set
channel ORA\_DISK\_1: specifying datafile(s) in backup set
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 15:59:07
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 15:59:08
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_nnndf\_TAG20120217T155907\_7mwybcbd\_.bkp tag=TAG20120217T155907 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2012/02/17 15:59:08

Starting Control File and SPFILE Autobackup at 2012/02/17 15:59:08
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775497548\_7mwybdnt\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 15:59:09
```

Including the archived logs at the same time, is possible too, by running the `backup tablespace tools plus archivelog delete input;` command. It isn't mandatory to delete the archived logs as part of the command, but it helps keep the FRA tidy.

### Backup Selected Data Files

With ARCHIVELOG mode enabled, you can even backup at the data file level.

Datafailes may be specified by id or my full name. The following backs up the SYSTEM tablespace's single datafile.

```sql
RMAN> report schema;
```
```text
Report of database schema for database with db\_unique\_name ANT12

List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    600      SYSTEM               \*\*\*     /srv/nffs/oradata/ant12/data/system01.dbf
...
12   500      XDB                  \*\*\*     /srv/nffs/oradata/ant12/data/xdb01.dbf
```
```sql
RMAN> backup datafile 1;
```
```text
Starting backup at 2012/02/17 16:12:10
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting full datafile backup set
channel ORA\_DISK\_1: specifying datafile(s) in backup set
input datafile file number=00001 name=/srv/nffs/oradata/ant12/data/system01.dbf
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 16:12:10
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 16:12:26
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_nnndf\_TAG20120217T161210\_7mwz2tyv\_.bkp tag=TAG20120217T161210 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:16
Finished backup at 2012/02/17 16:12:26

Starting Control File and SPFILE Autobackup at 2012/02/17 16:12:26
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775498346\_7mwz3bdl\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 16:12:27
```

You can also backup multiple data files together by specifying a comma separated list of id numbers or names:

```sql
RMAN> backup datafile
2> '/srv/nffs/oradata/ant12/data/tools01.dbf',
3> '/srv/nffs/oradata/ant12/data/users01.dbf';
```
```text
Starting backup at 2012/02/17 16:15:11
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting full datafile backup set
channel ORA\_DISK\_1: specifying datafile(s) in backup set
input datafile file number=00005 name=/srv/nffs/oradata/ant12/data/tools01.dbf
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 16:15:11
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 16:15:12
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_nnndf\_TAG20120217T161511\_7mwz8hvx\_.bkp tag=TAG20120217T161511 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2012/02/17 16:15:12

Starting Control File and SPFILE Autobackup at 2012/02/17 16:15:12
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775498513\_7mwz8k80\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 16:15:16
```

The above is equivalent to the command `backup datafile 5,6;` in the case of my database.

### Backup Archived Logs

You can take a backup of the archived logs at any time you wish simply by running the `backup archivelog all;` command as demonstrated at the beginning of this instalment.

It is possible to backup only a selection of archived logs, for example, a single log file:

```sql
RMAN> backup archivelog sequence 15;
```
```text
Starting backup at 2012/02/17 16:27:15
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting archived log backup set
channel ORA\_DISK\_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=15 RECID=71 STAMP=775499197
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 16:27:15
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 16:27:16
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_annnn\_TAG20120217T162715\_7mwzz3gk\_.bkp tag=TAG20120217T162715 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2012/02/17 16:27:16

Starting Control File and SPFILE Autobackup at 2012/02/17 16:27:16
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775499236\_7mwzz4w2\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 16:27:17
```

Or, if required, a number of them:

```sql
RMAN> backup archivelog sequence between 16 and 19;
```
```text
Starting backup at 2012/02/17 16:28:15
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting archived log backup set
channel ORA\_DISK\_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=16 RECID=72 STAMP=775499199
input archived log thread=1 sequence=17 RECID=73 STAMP=775499200
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 16:28:16
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 16:28:17
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/backupset/2012\_02\_17/o1\_mf\_annnn\_TAG20120217T162815\_7mx01041\_.bkp tag=TAG20120217T162815 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2012/02/17 16:28:17

Starting Control File and SPFILE Autobackup at 2012/02/17 16:28:17
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775499297\_7mx011ll\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 16:28:18
```

You may also specify the logs to be backed up using times. For example, the command `backup archivelog from time 'sysdate-7' until time 'sysdate-1';` will backup those archived logs, still on disc in the FRA, that were created within the time period specified.

And finally, for archived logs, if they have been stored in a section of the file system which is outside the FRA, you can still back those up.

If the archived logs are in the location `/srv/nffs/oradata/arch` then you back those up (to the FRA) by running the command `backup archivelog like '/srv/nffs/oradata/arch%';`

## Specifying a New Destination for Backups

In all of the preceding examples, the backups have been created in the FRA. What do you do if yo want to create a backup somewhere else?

The `format` parameter is how:

```sql
RMAN> backup tablespace users format='/media/oracle\_backups/ant12/%U';
```
```
Starting backup at 2012/02/17 16:51:52
using channel ORA\_DISK\_1
channel ORA\_DISK\_1: starting full datafile backup set
channel ORA\_DISK\_1: specifying datafile(s) in backup set
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA\_DISK\_1: starting piece 1 at 2012/02/17 16:51:52
channel ORA\_DISK\_1: finished piece 1 at 2012/02/17 16:51:53
piece handle=/media/oracle\_backups/ant12/2nn3ict8\_1\_1 tag=TAG20120217T165152 comment=NONE
channel ORA\_DISK\_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2012/02/17 16:51:53

Starting Control File and SPFILE Autobackup at 2012/02/17 16:51:54
piece handle=/srv/nffs/flashback\_area/ant12/ANT12/autobackup/2012\_02\_17/o1\_mf\_s\_775500714\_7mx1fbdd\_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/17 16:51:55
```

It can be seen from the _piece handle_ that the backup has indeed been sent to the non-FRA disc area.

The `format` parameter can be used for all types of backups, not just for tablespaces as in this example.
