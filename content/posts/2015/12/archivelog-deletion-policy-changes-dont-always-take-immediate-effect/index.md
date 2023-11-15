---
title: "Archivelog Deletion Policy Changes Don't Always Take Immediate Effect."
date: "2015-12-11"
categories: 
  - "oracle"
---

The standby database had the RMAN archivelog deletion policy set to 'NONE' instead of being 'APPLIED ON ALL STANDBY' and the FRA filled up to within an inch of its life, or 99% of its allocated quota! Not a major problem as this database was not in production, but still, an alert is an alert and has to be dealt with. However, things did not go quite as expected.

First things first, check the archivelog deletion policy on the standby database:

```
$ rman target / catalog user/password@catalog

RMAN> show archivelog deletion policy;
RMAN configuration parameters for database with db_unique_name xxxxxxxx are:
CONFIGURE ARCHIVELOG DELETION POLICY TO NONE;

RMAN> configure archivelog deletion policy to applied on all standby;

old RMAN configuration parameters:
CONFIGURE ARCHIVELOG DELETION POLICY TO NONE;
new RMAN configuration parameters:
CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON ALL STANDBY;
new RMAN configuration parameters are successfully stored

RMAN> exit 
```

As the FRA was currently at 99% usage, I expected Oracle to start clearing out archived logs that had been applied on the standby, in other words, the vast majority of them, Strangely, this didn't happen. Never mind, sometimes you have to encourage Oracle by transporting a new archived log from the primary, which usually kicks off the tidy up process. Let's do that, on the _primary_ database.

```bash
$ sqlplus / as sysdba
```

```sql
SQL> alter system archive log current;
System altered.

SQL> exit
```

Back on the standby database, tailing the alert log shows that the new archived log had arrived and been applied, but still no deletions. Hmmm, maybe a little more encouragement is required.

```bash
$ sqlplus / as sysdba
```

```sql
SQL> show parameter recovery_file_dest

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      +FRA
db_recovery_file_dest_size           big integer 40G
```

So, we have a quota of 40gb in the FRA for this standby, it's not yet in production, so that's a reasonable quota. How much is actually used? We know it's a lot - 99% - from the alert that started all this off, but let's check anyway.

```sql
SQL> set lines 300 pages 300 numw 15 trimspool on

SQL> col name format a10
SQL> col space_limit format 999,999,999,999
SQL> col space_used format 999,999,999,999
SQL> col space_reclaimable format 999,999,999,999
SQL> col number_of_files format 9,999,999

SQL> select * from v$recovery_file_dest;

NAME            SPACE_LIMIT       SPACE_USED SPACE_RECLAIMABLE NUMBER_OF_FILES          CON_ID
---------- ---------------- ---------------- ----------------- --------------- ---------------
+FRA         42,949,672,960   42,652,925,952    42,633,003,008           1,313               0
```

Pretty much, all of it, as expected. What makes up the usage? Is it _all_ archived logs or could there be restore points and/or flashback logs as well.

```sql
SQL> col percent_space_used format 990.00
SQL> col percent_space_reclaimable format 990.00

SQL> select * from v$flash_recovery_area_usage;

FILE_TYPE               PERCENT_SPACE_USED PERCENT_SPACE_RECLAIMABLE NUMBER_OF_FILES          CON_ID
----------------------- ------------------ ------------------------- --------------- ---------------
CONTROL FILE                          0.00                      0.00               0               0
REDO LOG                              0.00                      0.00               0               0
ARCHIVED LOG                         99.31                     99.26           1,313               0
BACKUP PIECE                          0.00                      0.00               0               0
IMAGE COPY                            0.00                      0.00               0               0
FLASHBACK LOG                         0.00                      0.00               0               0
FOREIGN ARCHIVED LOG                  0.00                      0.00               0               0
AUXILIARY DATAFILE COPY               0.00                      0.00               0               0
```

In this case, it's definitely archived logs and the vast majority can be reclaimed. Time to get rid! We know that gentle persuasion didn't have any effect, so let's increase the pressure a little:

```sql
SQL> alter system set db_recovery_file_dest_size = 35g scope=memory;
System altered.
```

And tailing the alert log, in another session, shows that things are finally happening.

```text
Fri Dec 11 14:03:13 2015
ALTER SYSTEM SET db_recovery_file_dest_size=35G SCOPE=MEMORY;
Fri Dec 11 14:03:13 2015
Deleted Oracle managed file +FRA/xxxxxxxx/ARCHIVELOG/2015_09_29/thread_1_seq_70.835.891673917
Deleted Oracle managed file +FRA/xxxxxxxx/ARCHIVELOG/2015_09_29/thread_1_seq_71.750.891675635
Deleted Oracle managed file +FRA/xxxxxxxx/ARCHIVELOG/2015_09_29/thread_1_seq_72.751.891675635
Deleted Oracle managed file +FRA/xxxxxxxx/ARCHIVELOG/2015_09_29/thread_1_seq_73.752.891675639
Deleted Oracle managed file +FRA/xxxxxxxx/ARCHIVELOG/2015_09_29/thread_1_seq_74.754.891680149
...
```

A few more gentle reductions, to avoid completely hanging things up, and we were eventually right down at 2gb FRA quota. Finally, I reset the quota back to the original 40gb and checked again:

```sql
SQL> alter system set db_recovery_file_dest_size = 40g scope=memory;
System altered.

SQL> select * from v$recovery_file_dest;

NAME            SPACE_LIMIT       SPACE_USED SPACE_RECLAIMABLE NUMBER_OF_FILES          CON_ID
---------- ---------------- ---------------- ----------------- --------------- ---------------
+FRA         42,949,672,960    2,112,880,640     2,092,957,696              61               0

SQL> select * from v$flash_recovery_area_usage;

FILE_TYPE               PERCENT_SPACE_USED PERCENT_SPACE_RECLAIMABLE NUMBER_OF_FILES          CON_ID
----------------------- ------------------ ------------------------- --------------- ---------------
CONTROL FILE                          0.00                      0.00               0               0
REDO LOG                              0.00                      0.00               0               0
ARCHIVED LOG                          4.92                      4.87              61               0
BACKUP PIECE                          0.00                      0.00               0               0
IMAGE COPY                            0.00                      0.00               0               0
FLASHBACK LOG                         0.00                      0.00               0               0
FOREIGN ARCHIVED LOG                  0.00                      0.00               0               0
AUXILIARY DATAFILE COPY               0.00                      0.00               0               0
```

Job done. With a _little_ encouragement!
