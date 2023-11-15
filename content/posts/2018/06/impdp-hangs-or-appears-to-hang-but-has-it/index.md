---
title: "IMPDP Hangs, or Appears to Hang - But Has it?"
date: "2018-06-07"
categories: 
  - "oracle"
---

You know the score, you are running an `impdp` and it looks to have hung up. You've watched the log file (or on screen messages) and it's sitting at something like:

```
Processing object type TABLE_EXPORT/TABLE/INDEX/INDEX
```

But hasn't moved from there for what seems hours. The alert log for the database is of no help, as there are no errors or warnings logged there. What's going on?

Is the import actually running? Check `DBA_DATAPUMP_JOBS` to find out:

```sql
select owner_name, job_name, operation, job_mode
from dba_datapump_jobs 
where state='EXECUTING' ;
```

Which gives something like:

```
OWNER_NAME JOB_NAME           OPERATION JOB_MODE
---------- ------------------ --------- --------
DUNBARNOR  SYS_IMPORT_FULL_01 IMPORT    FULL
```

So, we see at least one full import job is running. Good news. Do we have any sessions running though?

```sql
select owner_name, job_name, session_type 
from dba_datapump_sessions;
```

And we see this:

```
OWNER_NAME JOB_NAME           SESSION_TYPE
---------- ------------------ -------------
DUNBARNOR  SYS_IMPORT_FULL_01 DBMS_DATAPUMP
DUNBARNOR  SYS_IMPORT_FULL_01 MASTER
DUNBARNOR  SYS_IMPORT_FULL_01 WORKER
DUNBARNOR  SYS_IMPORT_FULL_01 WORKER
DUNBARNOR  SYS_IMPORT_FULL_01 WORKER
DUNBARNOR  SYS_IMPORT_FULL_01 WORKER
```

So, we have the master session, and a few workers. My job is running with `parallel=4` in the parameter file, so that's why there are 4 workers. Are they actually doing anything?

```sql
select v.status, v.sid,v.serial#,io.block_changes,event 
from v$sess_io io, v$session v 
where io.sid = v.sid 
and v.saddr in (
    select saddr 
    from dba_datapump_sessions
) order by sid;

STATUS SID  SERIAL# BLOCK_CHANGES EVENT
------ ---- ------- ------------- --------------------------------------------
ACTIVE 45   27      197679        PX Deq: Execute Reply 
ACTIVE 324  431     5484          wait for unread message on broadcast channel
ACTIVE 614  89      15406         wait for unread message on broadcast channel
ACTIVE 757  105     50130         wait for unread message on broadcast channel
ACTIVE 891  169     77216         wait for unread message on broadcast channel
ACTIVE 1035 59      76471         wait for unread message on broadcast channel
```

Hmm, looks like nothing is working at all. Every session appears to be waiting for something to be broadcast. The number of `BLOCK_CHANGES` should be increasing if the job was working correctly, shouldn't it?

The job has definitely hung.

Or has it?

Well, in this particular case, the clue is on screen, in the log file, and noted above.

The job is `Processing object type TABLE_EXPORT/TABLE/INDEX/INDEX` so, I'm hoping it's creating indexes. Can we check? Of course!

> For reasons of space across the page, I've had to `substr()` a couple of columns in the following SQL statement. You should refrain from doing so to get the full picture.

```sql
select s.sid, s.module, s.state, 
       substr(s.event, 1, 21) as event,
       s.seconds_in_wait as secs, 
       substr(sql.sql_text, 1, 30) as sql_text
from v$session s
join v$sql sql on sql.sql_id = s.sql_id
where s.module like 'Data Pump%'
order by s.module, s.sid;
```

And now we can see what's going on, The `SQL_TEXT` column shows that a number of parallel sessions are indeed creating indexes:

```
SID   MODULE            STATE    EVENT                  SECS SQL_TEXT
---- ---------------- ------- --------------------- ---- ------------------------------
614   Data Pump Master  WAITING  wait for unread messa  0    BEGIN :1 := sys.kupc$que_int.r
45    Data Pump Worker  WAITING  PX Deq: Execute Reply  64   CREATE INDEX "DUNBARNOR"."XIE1
192   Data Pump Worker  WAITING  direct path read temp  0    CREATE INDEX "DUNBARNOR"."XIE1
757   Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
757   Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
757   Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
857   Data Pump Worker  WAITING  direct path read temp  0    CREATE INDEX "DUNBARNOR"."XIE1
891   Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
891   Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
891   Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
1019  Data Pump Worker  WAITING  PX Deq: Execution Msg  64   CREATE INDEX "DUNBARNOR"."XIE1
1035  Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
1035  Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
1035  Data Pump Worker  WAITING  wait for unread messa  1    BEGIN :1 := sys.kupc$que_int.t
1324  Data Pump Worker  WAITING  PX Deq: Execution Msg  64   CREATE INDEX "DUNBARNOR"."XIE1
1749  Data Pump Worker  WAITING  direct path read temp  0    CREATE INDEX "DUNBARNOR"."XIE1
2034  Data Pump Worker  WAITING  PX Deq: Execution Msg  67   CREATE INDEX "DUNBARNOR"."XIE1
2153  Data Pump Worker  WAITING  direct path read temp  0    CREATE INDEX "DUNBARNOR"."XIE1
2177  Data Pump Worker  WAITING  PX Deq: Execution Msg  66   CREATE INDEX "DUNBARNOR"."XIE1
```

So, the `impdp` job _is_ still running and _is_ still working, it's just not _importing_ at the moment, it is building indexes.

Why does it appear hung? This is an exceedingly large table, with _far too many_ indexes for comfort. They all need to be recreated, so this takes time. Repeatedly executing the above query (the non-`substr()`'d version I mean) will show the names of the indexes changing every time it completes one index and moves on to the next. You will also see it moving on to a different table name when it has built all the indexes on the currently displayed table.

You will, I hope, also notice a number of `SID`s in the above output which are never mentioned in the preceding query results. This is why, I suspect, that there are a lot of hits on the web about the wait event `wait for unread message on broadcast channel` related to `impdp` (or `expdp`) but so far, none of those hits seem to go into any details about the sessions you don't find in `DBA_DATAPUMP_JOBS` or `DBA_DATAPUMP_SESSIONS`, perhaps a quick look in `V$SESSION` is more helpful when trying to track down suspected hangs in these utilities?

So, it turns out the job was not hung after all. At least, not in _this_ case.

Enjoy.
