---
title: "Oracle RMAN for Beginners - Part 1"
date: "2012-02-06T00:00:01Z"
categories: 
  - "oracle"
---

## Introduction

One of the things a DBA needs to be aware of, is RMAN. This has been around since Oracle 8 (or was it 8i?) and has been improving since then. It's almost pretty good at 11.2!

One of the things that I, as a DBA, need to get to grips with is RMAN. Most of the work I've been doing for the last few years have not involved very much in the way of RMAN usage. It's time I put an end to my almost complete ignorance.

I have set myself up a small test system where I can do my own RMAN training. I'll be writing up my results and findings as I go along. Feel free to correct me where I get things wrong - which I no doubt will!

## Sensible Prerequisites

I have found a few sensible prerequisites to using RMAN. For best results, set your environment up in a similar manner.

I'll be using a database named _ant12_ running on a server named _hubble_.

The first irritation I have with RMAN is it's use of dates. They are never expanded to how I like them to be. So run the following command in a shell session:

```bash
hubble> export NLS_DATE_FORMAT='yyyy/mm/dd hh24:mi:ss'
```

Next up, we have the problem that on some Linux/Unix systems the rman command is actually in two places, one in ORACLE_HOME, the other under the X11 installation. Usually, X11 is higher up `$PATH` than ORACLE_HOME, so we need to be sure we are using the right one.

```bash
hubble> alias rman='$ORACLE_HOME/bin/rman'
```

Using single quotes in the command allows the evaluation of ORACLE_HOME to be carried out at run time - when the `rman` command is called - not at define time. That way, whatever ORACLE_HOME is current at the time of running RMAN will be the correct one for the database.

Now we need to make sure that we have set the Oracle environment for the database to be backed up.

```bash
# I'll be using a database called ant12.
hubble> . oraenv
ORACLE_SID = [ant12] ? ant12
The Oracle base remains unchanged with value /srv/oracle 
```

Oracle advise that the database should be using a FLASH RECOVERY AREA (FRA), so let's set the database up to use one:

```bash
hubble> sqlplus / as sysdba
```
```sql
SQL> -- Commands must be typed in the following order ....
SQL> alter system set log_archive_duplex_dest='';
System altered.

SQL> alter system set log_archive_dest='';
System altered.

SQL> alter system set db_recovery_size=19G;
System altered.

SQL> alter system 
set db_recovery_dest='/srv/nffs/flashback_area/ant12';
System altered.
```

There is a reason for not having the archived logs stored within the FRA, it becomes a single point of failure. However, there are a few reasons for doing so, the main one being that you have all files necessary for a recovery in the same place.

If you are using the FRA for archived logs, it is probably a good idea to be mirroring the FRA.

For my test system, I _am_ putting the archived logs into the FRA.

```sql
SQL> alter system set
log_archive_dest_1='LOCATION=USE_DB_RECOVERY_FILE_DEST';
System altered.

SQL> alter system set log_archive_dest_state_1=enable;
System altered.
```

## Check and Amend Configuration

RMAN comes configured with reasonably sensible defaults. However, they are not as desired for my use, so the first thing to do is look at the default settings, and change them.

```bash
# Run the correct version of RMAN for the database
#the display the current default settings:
hubble> rman target /
```
```sql
RMAN> show all;
```
```text
RMAN configuration parameters for database with db_unique_name ANT12 are:
CONFIGURE RETENTION POLICY TO REDUNDANCY 1; # default
CONFIGURE BACKUP OPTIMIZATION OFF; # default
CONFIGURE DEFAULT DEVICE TYPE TO DISK; # default
CONFIGURE CONTROLFILE AUTOBACKUP OFF; # default
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F'; # default
CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO BACKUPSET; # default
CONFIGURE DATAFILE BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
CONFIGURE ARCHIVELOG BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
CONFIGURE MAXSETSIZE TO UNLIMITED; # default
CONFIGURE ENCRYPTION FOR DATABASE OFF; # default
CONFIGURE ENCRYPTION ALGORITHM 'AES128'; # default
CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE ; # default
CONFIGURE ARCHIVELOG DELETION POLICY TO NONE; # default
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/srv/oracle/product/11gR1/db/dbs/snapcf_ant12.f'; # default
```

We are advised, by Oracle, that we should configure RMAN to always backup the control file (which includes the spfile - if one is in use) if we are not using an RMAN catalogue. This is because without a catalogue we store details of backups in the control file, so we don't want to lose those.

We will also configure RMAN to prevent the copying of virgin blocks. These are blocks in data files which have _never_ been used at all. (Being used and then emptied means a block is no longer virgin.) This can help to reduce backup times.

We will do all of them in one go, using a `run {}` block.

```sql
RMAN> run {
2> configure controlfile autobackup on;
3> configure backup optimization on;
4> }
```
```text
new RMAN configuration parameters:
CONFIGURE CONTROLFILE AUTOBACKUP ON;
new RMAN configuration parameters are successfully stored

new RMAN configuration parameters:
CONFIGURE BACKUP OPTIMIZATION ON;
new RMAN configuration parameters are successfully stored
```

We are ready to backup the ant12 database to the FRA.
