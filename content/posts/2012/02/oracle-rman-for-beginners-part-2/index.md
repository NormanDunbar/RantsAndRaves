---
title: "Oracle RMAN for Beginners - Part 2"
date: "2012-02-06T00:00:03Z"
categories: 
  - "oracle"
---

At the end of [Oracle RMAN for Beginners - Part 1](http://qdosmsq.dunbar-it.co.uk/blog/2012/02/oracle-rman-for-beginners-part-1/ "Oracle RMAN for Beginners – Part 1") I was ready to begin a backup of the ant12 database. Read on ...

## Cold Backup - Backupset Type

There are two different types of cold backup. The first uses RMAN's own internal format for the dump files - a _backupset_ - the other uses _image copies_ of the individual database files. In this part I'll concentrate on a backuset type of cold backup. I'll look at image copies later.

A cold backup is always a _full_ backup. To take a cold backup the database needs to be MOUNTed. This is slightly different from a normal cold backup using the OS tools to copy files, as the database should be SHUTDOWN in that case.

A cold backup is all that you can do when the database is running in NOARCHIVELOG mode, however, you can take a cold backup of a database running in ARCHIVELOG mode using the commands shown below.

```sql
RMAN> connect target /
connected to target database: ANT12 (DBID=2799264292)
```
```sql
RMAN> shutdown;
database dismounted
Oracle instance shut down
```
```sql
RMAN> startup mount;
connected to target database (not started)
Oracle instance started
database mounted
...
```

The following command creates a new backupset in the FRA. The files will be written to the FRA in a directory named _backupset/SID_.

```sql
RMAN> backup full database;
```
```text
Starting backup at 2012/02/06 09:58:40
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00010 name=/srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01_01.dbf
...
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting piece 1 at 2012/02/06 09:58:41
channel ORA_DISK_1: finished piece 1 at 2012/02/06 09:59:16
piece handle=/srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T095841_7lz92krk_.bkp tag=TAG20120206T095841 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:35
Finished backup at 2012/02/06 09:59:16

Starting Control File and SPFILE Autobackup at 2012/02/06 09:59:16
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_06/o1_mf_s_774524380_7lz93qbg_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/06 09:59:23
```

And that's all there is to it. The backup file(s) will be created in the FRA as detailed above by various _piece handle_ messages. You can see the backup details using the `list backup` command:

```sql
RMAN> list backup;
```
```text
List of Backup Sets
===================

BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
6       Full    365.34M    DISK        00:00:35     2012/02/06 09:59:16
        BP Key: 6   Status: AVAILABLE  Compressed: NO  Tag: TAG20120206T095841
        Piece Name: /srv/nffs/flashback_area/ant12/ANT12/backupset/2012_02_06/o1_mf_nnndf_TAG20120206T095841_7lz92krk_.bkp
  List of Datafiles in backup set 6
  File LV Type Ckp SCN    Ckp Time            Name
  ---- -- ---- ---------- ------------------- ----
  1       Full 847342     2012/02/06 09:39:40 /srv/nffs/oradata/ant12/data/system01.dbf
...
  12      Full 847342     2012/02/06 09:39:40 /srv/nffs/oradata/ant12/data/xdb01.dbf

BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
7       Full    9.70M      DISK        00:00:03     2012/02/06 09:59:20
        BP Key: 7   Status: AVAILABLE  Compressed: NO  Tag: TAG20120206T095916
        Piece Name: /srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_06/o1_mf_s_774524380_7lz93qbg_.bkp
  SPFILE Included: Modification time: 2012/02/06 09:40:06
  SPFILE db_unique_name: ANT12
  Control File Included: Ckp SCN: 847342       Ckp time: 2012/02/06 09:39:40
```

The backup will be created in _DB_RECOVERY_DEST_/SID/backupset. A separate backup took place to copy the controlfile and spfile so that details of this current backup remain safe.

{{< alert theme="warning" >}}
In most cases the SID will be used to determine the output directory in the FRA as specified by the database parameter `DB_RECOVERY_DEST`. However it is not ORACLE_SID that is actually used but the `DB_UNIQUE_NAME` initialisation parameter. I used SID as it's easier to type!
{{< /alert >}}

It is possible to avoid backing up to the FRA by specifying a `FORMAT` parameter:

```sql
RMAN> backup format='/media/oracle_backups/ant12/ant12_backupset.rman' full database;
```
```text
Starting backup at 2012/02/06 12:30:20
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00010 name=/srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01_01.dbf
...
input datafile file number=00006 name=/srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: starting piece 1 at 2012/02/06 12:30:20
channel ORA_DISK_1: finished piece 1 at 2012/02/06 12:31:45
piece handle=/media/oracle_backups/ant12/ant12_backupset.rman tag=TAG20120206T123020 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:01:25
Finished backup at 2012/02/06 12:31:45

Starting Control File and SPFILE Autobackup at 2012/02/06 12:31:46
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_06/o1_mf_s_774524380_7lzl1lcg_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/06 12:31:47
```

Unfortunately, if you look carefully at the last few lines of output, the controlfile and spfile autobackup have _still_ gone to the FRA and not to the desired location. What to do?

You can temporarily turn off the controlfile autobackup, run a database backup followed by a manual controlfile (and spfile) backup and then turn controlfile autobackup back on, as follows. You don't have to use a `run {}` block as RMAN will happily accept all the commands separately. (This is not always the case!)

```sql
RMAN> run {
2> configure controlfile autobackup off;
3> backup format = '/media/oracle_backups/ant12/backupset_4.rman' full database;
4> backup format = '/media/oracle_backups/ant12/cf_backup_manual.f' current controlfile;
5> configure controlfile autobackup on;
}
```
```text
old RMAN configuration parameters:
CONFIGURE CONTROLFILE AUTOBACKUP ON;
new RMAN configuration parameters:
CONFIGURE CONTROLFILE AUTOBACKUP OFF;
new RMAN configuration parameters are successfully stored

Starting backup at 2012/02/06 15:46:06
using channel ORA_DISK_1
...
piece handle=/media/oracle_backups/ant12/backupset_4.rman tag=TAG20120206T154606 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:25
Finished backup at 2012/02/06 15:46:32

Starting backup at 2012/02/06 15:46:32
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current control file in backup set
channel ORA_DISK_1: starting piece 1 at 2012/02/06 15:46:33
channel ORA_DISK_1: finished piece 1 at 2012/02/06 15:46:40
piece handle=/media/oracle_backups/ant12/cf_backup_manual.f tag=TAG20120206T154632 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:07
Finished backup at 2012/02/06 15:46:40

old RMAN configuration parameters:
CONFIGURE CONTROLFILE AUTOBACKUP OFF;
new RMAN configuration parameters:
CONFIGURE CONTROLFILE AUTOBACKUP ON;
new RMAN configuration parameters are successfully stored
```

There has to be an easier way to do this doesn't there? What about when you don't know for sure that the configuration parameter is on or off? I'm sure there's a way, and if there is, I'll find it.

**UPDATE** there is a way. Simply use the `set` command instead of `configure`. For example, the above should be replaced by:

```sql
RMAN> run {
2> set controlfile autobackup off;
3> backup format = '/media/oracle_backups/ant12/backupset_4.rman' full database;
4> backup format = '/media/oracle_backups/ant12/cf_backup_manual.f' current controlfile;
}
```

The `set` remains in force until the end of the session, or, as in this case, until the end of the `run` block.

Anyway, you now have a cold backup of the database and a safety copy of the controlfile. Unfortunately, perhaps, the backup of the controlfile didn't take a backup of the spfile. To do that you need to add in a separate manual copy of the spfile similar to the following:

```sql
RMAN> backup format = '/media/oracle_backups/ant12/sp_backup_manual.f' spfile;
```
```text
Starting backup at 2012/02/06 15:58:31
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current SPFILE in backup set
channel ORA_DISK_1: starting piece 1 at 2012/02/06 15:58:31
channel ORA_DISK_1: finished piece 1 at 2012/02/06 15:58:32
piece handle=/media/oracle_backups/ant12/sp_backup_manual.f tag=TAG20120206T155831 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2012/02/06 15:58:32

Starting Control File and SPFILE Autobackup at 2012/02/06 15:58:33
piece handle=/srv/nffs/flashback_area/ant12/ANT12/autobackup/2012_02_06/o1_mf_s_774524380_7lzy59go_.bkp comment=NONE
Finished Control File and SPFILE Autobackup at 2012/02/06 15:58:34
```

Did you notice? The backup of the spfile includes a backup of the controlfile. Is nothing consistent with RMAN? ;-)

Coming soon, restoring and recovering a database from a cold backup.
