---
title: "Asmcmd or ASM Instance Backups or Queries Hang"
date: "2015-04-17"
categories: 
  - "oracle"
---

Sometimes an ASM instance hangs for _no apparent reason_ and this causes problems when backing up the ASM Metadata. Running queries against `V$ASM_DISK` and similar views may also hang. This blog post should go some way to helping diagnose the problem, and providing a fix.

ASM metadata backups on a couple of our servers had been failing, the backups were run from a system called CommVault and the job scheduler there showed that they simply sat at 0% forever, or would have if we allowed them! There were no error messages or codes to speak of - the job simply sits in CommVault at 0% and never ends.

This was tracked down to the `md_backup` command being run from `asmcmd`. Running the command manually also just hung and the session had to be killed to release it.

Using sqlplus on the ASM instance and attempting to query `V$ASM_DISK` or other similar ASM views, also hung.

Looking at `V$SESSION` in the ASM instance, with the following query shows the problem:

```sql
set lines 350 trimspool on pages 300

select sid, state, event, seconds_in_wait, blocking_session
from   v$session
where  blocking_session is not null
or sid in (select blocking_session 
           from   v$session 
           where  blocking_session is not null)
order by sid;

       SID STATE    EVENT                 SECONDS_IN_WAIT BLOCKING_SESSION
---------- -------- --------------------- --------------- ----------------
        15 WAITING  enq: DD - contention            73683              254
        16 WAITING  enq: DD - contention            15692              254
        17 WAITING  enq: DD - contention           117109              254
        93 WAITING  enq: DD - contention            61107              254
        95 WAITING  enq: DD - contention           242327              254
        96 WAITING  enq: DD - contention            68731              254
       167 WAITING  GPnP Get Item                 2471652
       172 WAITING  enq: DD - contention           117109              254
       173 WAITING  enq: DD - contention           147026              254
       176 WAITING  enq: DD - contention            37787              254
       177 WAITING  enq: DD - contention           658138              254
       178 WAITING  enq: DD - contention            42238              254
       251 WAITING  enq: DD - contention           315711              254
       253 WAITING  enq: DD - contention            97075              254
       254 WAITING  rdbms ipc reply                     0              167
       255 WAITING  enq: DD - contention             4140              254
       257 WAITING  enq: DD - contention           521537              254

17 rows selected.
```

Almost all hung sessions are waiting for sid 254. Sid 254 is itself waiting on 167 which is not waiting on a session, but on the `GPnP Get Item` event.

A search of MOS shows that this is caused by an unpublished bug. Note 1375505.1 which mentions killing the `gpnpd.bin` process with a HUP, which will cause it to immediately restart, refers the reader to note 1392934.1 for full details. That latter note simply says:

```bash
kill -HUP 
```

Full details indeed! There's not even a pid to be killed.

In our specific case, the following was required:

```bash
ps -ef | grep -i g\[p\]npd

grid      4084     1  0  Jul 15  ?        04:37:48 /app/gridsoft/11.2.0.3/bin/gpnpd.bin

su - grid
Password: ******

kill -HUP 4084
```

It can be seen that this is safe, according to Oracle, and the `gpnpd.bin` process will be automatically restarted - even on production systems!

```bash
ps -ef | grep -i g\[p\]npd

grid     19015     1 14 09:23:10 ?        00:00:00 /app/gridsoft/11.2.0.3/bin/gpnpd.bin
```

It can be seen from the above that the daemon is running and has a new pid and start time. If we check in the database again, there will be no waiting sessions and the ASM Metadata backups will work, as will querying `V$ASM_DISK` etc.
