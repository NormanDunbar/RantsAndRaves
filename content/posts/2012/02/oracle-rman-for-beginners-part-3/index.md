---
title: "Oracle RMAN for Beginners – Part 3"
date: "2012-02-06T00:00:05Z"
categories: 
  - "oracle"
---

At the end of [Oracle RMAN for Beginners – Part 2](http://qdosmsq.dunbar-it.co.uk/blog/2012/02/oracle-rman-for-beginners-part-2/ "Oracle RMAN for Beginners – Part 2") I had created a few backups of the ant12 database. In this part, I explore how to trash a database and recover from that trashing using the cold backups taken.

Bear in mind that any work done after the cold backup was taken will be lost. With a cold backup, you cannot roll forward to re-apply archived logs etc because when the backup was taken, the database was in a consistent state.

{{< alert theme="danger" >}}
A cold backup, taken of a database running in archivelog mode may still require archived logs to be present to restore the backup. Document 337450.1 on My Oracle Support has the details. This affects databases running RMAN version 9.2.0.1 through 10.2.0.5.
 
This could explain why I'm not seeing it as I'm restoring an 11.2 RMAN backup.
 
Thanks to David Farkough (@Farkough) for this information.
{{< /alert >}}

## Cold Recovery – Backupset Type

In order to restore a database I first need a broken database. To see what files I need to destroy, I can use the RMAN command `report schema` as follows:

```sql
RMAN> report schema;
```
```text
Report of database schema for database with db_unique_name ANT12

List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    600      SYSTEM               ***     /srv/nffs/oradata/ant12/data/system01.dbf
2    500      SYSAUX               ***     /srv/nffs/oradata/ant12/data/sysaux01.dbf
3    512      UNDOTBS1             ***     /srv/nffs/oradata/ant12/data/undotbs01.dbf
4    700      PERFSTAT             ***     /srv/nffs/oradata/ant12/data/perfstat01_01.dbf
5    10       TOOLS                ***     /srv/nffs/oradata/ant12/data/tools01.dbf
6    10       USERS                ***     /srv/nffs/oradata/ant12/data/users01.dbf
7    300      AUDIT01              ***     /srv/nffs/oradata/ant12/data/audit01_01.dbf
8    1024     NLWLDELFTFEWSDAT01   ***     /srv/nffs/oradata/ant12/data/NLWLDELFTFEWSMCDat01_01.dbf
9    350      NLWLDELFTFEWSIDX01   ***     /srv/nffs/oradata/ant12/data/NLWLDELFTFEWSMCIdx01_01.dbf
10   3224     NLWLDELFTFEWSLOB01   ***     /srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01_01.dbf
11   128      UTILITY01            ***     /srv/nffs/oradata/ant12/data/utility01_01.dbf
12   500      XDB                  ***     /srv/nffs/oradata/ant12/data/xdb01.dbf

List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1    200      TEMP                 200         /srv/nffs/oradata/ant12/data/temp01.dbf
```

I shall start simple, and destroy the TOOLS tablespace by deleting the file.

```sql
RMAN> shutdown;
database dismounted
Oracle instance shut down
```
```sql
RMAN> host "mv /srv/nffs/oradata/ant12/data/tools01.dbf /srv/nffs/oradata/ant12/data/tools01.dbf.old";
host command complete
```
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
RMAN-03002: failure of startup command at 02/06/2012 16:14:41
ORA-01157: cannot identify/lock data file 5 - see DBWR trace file
ORA-01110: data file 5: '/srv/nffs/oradata/ant12/data/tools01.dbf'
```
You will note that the database didn't open, but is stuck in a `MOUNT`ed state. This is required to carry out a recovery. Lets restore the cold backup.

```sql
RMAN> connect target /
connected to target database: ANT12 (DBID=2799264292, not open)
```
```sql
RMAN> restore database;
```
```text
Starting restore at 2012/02/06 16:15:49
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK
...
skipping datafile 1; already restored to file /srv/nffs/oradata/ant12/data/system01.dbf
skipping datafile 2; already restored to file /srv/nffs/oradata/ant12/data/sysaux01.dbf
skipping datafile 3; already restored to file /srv/nffs/oradata/ant12/data/undotbs01.dbf
skipping datafile 4; already restored to file /srv/nffs/oradata/ant12/data/perfstat01_01.dbf
skipping datafile 6; already restored to file /srv/nffs/oradata/ant12/data/users01.dbf
skipping datafile 7; already restored to file /srv/nffs/oradata/ant12/data/audit01_01.dbf
skipping datafile 8; already restored to file /srv/nffs/oradata/ant12/data/NLWLDELFTFEWSMCDat01_01.dbf
skipping datafile 9; already restored to file /srv/nffs/oradata/ant12/data/NLWLDELFTFEWSMCIdx01_01.dbf
skipping datafile 10; already restored to file /srv/nffs/oradata/ant12/data/NLWLDELFTFEWSModDat01_01.dbf
skipping datafile 11; already restored to file /srv/nffs/oradata/ant12/data/utility01_01.dbf
skipping datafile 12; already restored to file /srv/nffs/oradata/ant12/data/xdb01.dbf
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00005 to /srv/nffs/oradata/ant12/data/tools01.dbf
channel ORA_DISK_1: reading from backup piece /media/oracle_backups/ant12/backupset_4.rman
channel ORA_DISK_1: piece handle=/media/oracle_backups/ant12/backupset_4.rman tag=TAG20120206T154606
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:07
Finished restore at 2012/02/06 16:16:00
```
```sql
RMAN> alter database open;
database opened
```

How easy was that then?

What happens if I've lost the SYSTEM tablespace then? Let's see:

```sql
RMAN> shutdown
database closed
database dismounted
Oracle instance shut down
```
```sql
RMAN> host "mv /srv/nffs/oradata/ant12/data/system01.dbf /srv/nffs/oradata/ant12/data/system01.dbf.old";
host command complete
```
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
RMAN-03002: failure of startup command at 02/06/2012 16:20:45
ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
ORA-01110: data file 1: '/srv/nffs/oradata/ant12/data/system01.dbf'
```

As before, we have "lost" a data file, this time making up the SYSTEM tablespace. Can we recover from this? Of course!

```sql
RMAN> restore database;
```
```text
Starting restore at 2012/02/06 16:39:43
using channel ORA_DISK_1

skipping datafile 2; already restored to file /srv/nffs/oradata/ant12/data/sysaux01.dbf
...
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /srv/nffs/oradata/ant12/data/system01.dbf
channel ORA_DISK_1: reading from backup piece /media/oracle_backups/ant12/backupset_3.rman
channel ORA_DISK_1: piece handle=/media/oracle_backups/ant12/backupset_3.rman tag=TAG20120206T153851
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:02:36
Finished restore at 2012/02/06 16:42:20
```

And now I can open up the database again for use, or can I?

```sql
RMAN alter database open;
```
```text
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of startup command at 02/06/2012 16:43:49
ORA-01113: file 1 needs media recovery
ORA-01110: data file 1: '/srv/nffs/oradata/ant12/data/system01.dbf'
```

It appears not! Even though I've recovered the SYSTEM tablespace file(s) I still have potential updates etc in the online redo logs. A quick `recover database` command should resolve any issues:

```sql
RMAN> recover database;
```
```text
Starting recover at 2012/02/06 16:45:29
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:02

Finished recover at 2012/02/06 16:45:32
```

Now, I should be able to open up the database:

```sql
RMAN> alter database open;
database opened
```

Job done. All recovered and usable.

It is not possible to recover a single data file, or tablespace, from a cold backup if the database has been updated in any way since the cold backup was taken _unless_ the database is running in ARCHIVELOG mode.

If the database is in NOARCHIVELOG mode, then all you can do is restore the complete database. The [next part](/posts/2012/02/oracle-rman-for-beginners-part-4/ "Oracle RMAN for Beginners – Part 4") gives a demonstration of this in action.

In ARCHIVELOG mode, RMAN will notice if any of the files have been updated, as shown in the following example where changes have been made to the database since the cold backup and "suddenly" a data file goes missing.

In this example, the user _norman_ created a new table named _test_ in the _users_ tablespace, and added a single row to it. Then the file making up the users tablespace was renamed.

```sql
RMAN> startup;
```
```text
connected to target database (not started)
Oracle instance started
database mounted
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of startup command at 02/06/2012 20:26:48
ORA-01157: cannot identify/lock data file 6 - see DBWR trace file
ORA-01110: data file 6: '/srv/nffs/oradata/ant12/data/users01.dbf'
```

As we are running in archivelog mode, we can recover only the users tablespace from the cold backup, and using the archive logs, recover the transactions.

```sql
RMAN> restore tablespace users;
```
```text
Starting restore at 2012/02/06 20:28:40
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK
...
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00006 to /srv/nffs/oradata/ant12/data/users01.dbf
channel ORA_DISK_1: reading from backup piece /media/oracle_backups/ant12/backupset_4.rman
channel ORA_DISK_1: piece handle=/media/oracle_backups/ant12/backupset_4.rman tag=TAG20120206T154606
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:07
Finished restore at 2012/02/06 20:28:49
```
```sql
RMAN> recover tablespace users;
```
```text
Starting recover at 2012/02/06 20:29:16
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:00

Finished recover at 2012/02/06 20:29:16
```
```sql
RMAN> alter database open;
database opened
```

That saves time as all we had to do was recover a single tablespace, which in this case consists of only one data file, and then reapply the committed transactions from the archivelogs which, handily, were still online. Had they not been online, RMAN would have found them in a backup and used those to recover the database.

This, of course, relies on a complete set of archived logs being found either online or in a backup. If any are missing, the database cannot be recovered fully.

## Restoring a Cold Backup With a Newer Controlfile

Be aware that if your cold backup is "of some age" and the database has moved on since it was taken, restoring the database back to the cold backup will work fine, and most normally, give no errors. However, when you try to opoen the database, you will be told that one or other file _wasn't restored from an old enough backup_.

An intriguing error message indeed, plus, the database will not open. In this situation, your contolfile is too new! All you have to do is as follows:

- Find the most recent backup of the controlfile that occurred prior to the cold backup that you have just restored. `List backup of controlfile` will suffice.
- Note the Handle - it will be something like "c-DBID-yyyymmdd-nn" where DBID is the database ID and yyyymmdd-nn is the date and a sequence number.
- Shut down the database, then startup nomount.
- Restore controlfile from "c-DBID-yyyymmdd-nn"

You now have the database files restored and the controlfile suitable for use, also restored. You can now open the database with `alter database open resetlogs`.
