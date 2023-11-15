---
title: "Oracle RMAN for Beginners – Part 4"
date: "2012-02-06T00:00:07Z"
categories: 
  - "oracle"
---

So far I have managed to dump and recover a database running in ARCHIVELOG mode. That is the most sensible mode for a production database and will be the case for the rest of this small RMAN guide. However, what if your databases are not running in ARCHIVELOG mode? What can RMAN do for you?

## Running a Database in NOARCHIVELOG Mode

The database is shutdown at the moment, so we will MOUNT it using RMAN and create a brand new full cold backup in the FRA.

```sql
RMAN> connect target /
connected to target database (not started)

RMAN> startup mount

Oracle instance started
database mounted
...
```

```sql
RMAN> backup full database;
```
```text
Starting backup at 2012/02/06 20:47:41
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00010 name=/srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01_01.dbf
...
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting piece 1 at 2012/02/06 20:47:43
channel ORA_DISK_1: finished piece 1 at 2012/02/06 20:48:28
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T204742_7m0h3hmw_.bkp tag=TAG20120206T204742 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:45
Finished backup at 2012/02/06 20:48:28

Starting Control File and SPFILE Autobackup at 2012/02/06 20:48:28
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_06/o1_mf_s_774563524_7m0h4xng_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/06 20:48:31
```
```sql
RMAN> startup;
database is already started
database opened
```

Again, we will trash the database by renaming a data file, after doing some work in a SQL*Plus session. The session to attempt a recovery using RMAN went like this:

```sql
RMAN> startup
```
```text
connected to target database (not started)
Oracle instance started
database mounted
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of startup command at 02/06/2012 21:00:45
ORA-01157: cannot identify/lock data file 6 - see DBWR trace file
ORA-01110: data file 6: '/srv/nffs/oradata/ant12/data/users01.dbf'
```

From the filename, I know it's the users tablespace that's affected, so I will attempt to restore and recover just that:

```sql
RMAN> restore tablespace users;
```
```text
Starting restore at 2012/02/06 21:01:21
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00006 to /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T204742_7m0h3hmw_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T204742_7m0h3hmw_.bkp tag=TAG20120206T204742
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 2012/02/06 21:01:23
```

It looks like RMAN allows a single tablespace to be restored, will it recover?

```sql
RMAN> recover tablespace users;
```
```text
Starting recover at 2012/02/06 21:01:44
using channel ORA_DISK_1

starting media recovery
...
RMAN-08187: WARNING: media recovery until SCN 855319 complete
Finished recover at 2012/02/06 21:01:46
```

That warning doesn't look too healthy. I next attempt to open the database:

```sql
RMAN> startup;
```
```text
database is already started
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of startup command at 02/06/2012 21:02:09
ORA-01113: file 6 needs media recovery
ORA-01110: data file 6: '/srv/nffs/oradata/ant12/data/users01.dbf'
```

So, that's it, with a database running in NOARCHIVELOG mode, you lose data if the changes that you are trying to recover have aged out of the online redo logs. Had they still been there I would have been able to recover the tablespace and open the database, as it is, I now need to do a full restore to get a consistent database back up and running. The database will be consistent, but my changes since the last full backup will be lost. Mine and everyone elses of course.

```sql
RMAN> restore database;
```
```text
Starting restore at 2012/02/06 21:08:11
using channel ORA_DISK_1

skipping datafile 6; already restored to file /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /srv/nffs/oradata/ant12/data/system01.dbf
...
channel ORA_DISK_1: restoring datafile 00012 to /srv/nffs/oradata/ant12/data/xdb01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T204742_7m0h3hmw_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T204742_7m0h3hmw_.bkp tag=TAG20120206T204742
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:03:56
Finished restore at 2012/02/06 21:12:08
```
```sql
RMAN> sql "alter database open resetlogs";
sql statement: alter database open resetlogs
```

That's it then, the database has been restored and opened.

One other thing, RMAN is smart enough to notice that my unsuccessful attempt to restore just the users tablespace meant that it didn't have to restore it again. That saved a little time.

The moral to this little exercise is simple. Don't run your databases in NOARCHIVELOG mode because you will lose data. In addition, you must always have downtime to backup the database. If you attempt to backup the database while it is open, you will see the following RMAN error message:

```sql
RMAN> backup full database;
```
```text
Starting backup at 2012/02/07 07:19:35
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03009: failure of backup command on ORA_DISK_1 channel at 02/07/2012 07:19:36
ORA-19602: cannot backup or copy active file in NOARCHIVELOG mode
```

## Recovery From a Total Loss

Even in NOARCHIVEMODE and without using a catalogue, it is possible to recover a database when the controlfiles are lost or unusable. However, this is only possible, according to the manual, from a controlfile autobackup. You also must have the database unique identifier (DBID) to hand as well as the configured format of the controlfile autobackup files.

In the following example, I made sure that I had a backup of the database, the DBID (which RMAN helpfully displays when you connect to the target at the `RMAN> prompt`) and my controlfile autobackup format is the default, so I didn't have to worry about that.

I shut down the database and deleted all the files. That represents a total failure scenario. The database is running in NOARCHIVELOG by the way. Let's recover it.

```sql
RMAN> startup nomount

connected to target database (not started)
Oracle instance started
...
```

The database cannot be mounted as we have no controlfiles.

```sql
RMAN> set DBID=2799264292
executing command: SET DBID
```

This allows RMAN to try and find the controlfile autobackup.

```sql
RMAN> restore controlfile from autobackup;
```
```text
Starting restore at 2012/02/07 08:21:39
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=19 device type=DISK

recovery area destination: /srv/nffs/flashback_area/ant12
database name (or database unique name) used for search: ANT12
channel ORA_DISK_1: AUTOBACKUP /srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_07/o1_mf_s_774602851_7m1noocb_.bkp found in the recovery area
channel ORA_DISK_1: looking for AUTOBACKUP on day: 20120207
channel ORA_DISK_1: restoring control file from AUTOBACKUP /srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_07/o1_mf_s_774602851_7m1noocb_.bkp
channel ORA_DISK_1: control file restore from AUTOBACKUP complete
output file name=/srv/nffs/oradata/ant12/ctrl/control01.ctl
output file name=/srv/nffs/flashback_area/ant12/ctrl/control02.ctl
output file name=/srv/nffs/oradata/ant12/ctrl/control03.ctl
Finished restore at 2012/02/07 08:21:40
```

So far so good. I have my controlfiles back. As I was using the default autobackup format, I didn't have to set the format, however, if I had changed the format at some point, I need to tell RMAN what it is. In that case, the above restore would look like the following:

```sql
RMAN> set DBID=2799264292
executing command: SET DBID

RMAN> run {
2> set controlfile autobackup format for device type disk to 'your format here';
3> restore controlfile from autobackup;
4> } 
```

The end result would be, hopefully, the restoration of the latest controlfile backup.

The controlfiles are back, but I'm are still without the data files.

**Note**: _any_ time that you have to restore the controlfile as part of a restore means that you must open the database with the `resetlogs` option.

```sql
RMAN alter database mount;
database mounted
```
```sql
RMAN> restore database;
```
```text
Starting restore at 2012/02/07 08:40:20
Starting implicit crosscheck backup at 2012/02/07 08:40:20
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=19 device type=DISK
Crosschecked 1 objects
Finished implicit crosscheck backup at 2012/02/07 08:40:21

Starting implicit crosscheck copy at 2012/02/07 08:40:21
using channel ORA_DISK_1
Crosschecked 17 objects
Finished implicit crosscheck copy at 2012/02/07 08:40:22

searching for all files in the recovery area
cataloging files...
cataloging done

List of Cataloged Files
=======================
File Name: /srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_07/o1_mf_s_774602851_7m1noocb_.bkp

using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /srv/nffs/oradata/ant12/data/system01.dbf
...
channel ORA_DISK_1: restoring datafile 00012 to /srv/nffs/oradata/ant12/data/xdb01.dbf
channel ORA_DISK_1: reading from backup piece /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_07/o1_mf_nnndf_TAG20120207T072806_7m1nn6rq_.bkp
channel ORA_DISK_1: piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_07/o1_mf_nnndf_TAG20120207T072806_7m1nn6rq_.bkp tag=TAG20120207T072806
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:03:38
Finished restore at 2012/02/07 08:44:03
```
```sql
RMAN> alter database open resetlogs;
database opened
```

Now that's what I call magic! ;-)
