---
title: "Statspack Snapshot Fails ORA-01400 Cannot Insert NULL ..."
date: "2013-05-01"
categories: 
  - "linux"
  - "oracle"
---

Oh hum. An 11.2.0.3 Enterprise Edition production database has statspack taking a regular snapshot under the control of a dbms_scheduler job. For no apparent reason, the snapshot started failing with _ORA-01400 Cannot insert NULL into PERFSTAT.STATS$SYSTEM_EVENT.EVENT_. This was an interesting one to fix.

The following is the investigative process, in brief.

- Test the snapshot process with a manual one - same error.
- Google and My Oracle Support aka MOS, were no help whatsoever. I was on my own! Twitter was useful and Noons ([@wizofoz2k](https://twitter.com/wizofoz2k "Noon's Twitter profile")) suggested a statspack mismatch could cause this error.
- Knowing that I had reinstalled statspack on this database a few weeks ago led me to drop and reinstall statspack and to recreate the jobs required to take regular snapshots and to purge old data. No joy, same problem.
- Hunt down the code in `V$SQL` to see what's going on here. A quick script helped out:

    ```sql
    SELECT sql_fulltext
    FROM   v$sql
    WHERE  DBMS_LOB.INSTR(sql_fulltext, 'SYSTEM_EVENT') <> 0
    AND    DBMS_LOB.INSTR(sql_fulltext, 'INSERT') <> 0;
    ```   
    That showed an insert statement, as expected, reading the `EVENT` column from `V$SYSTEM_EVENT` – which, given half a brain, makes sense! I didn't have half a brain at the time – as will become obvious!
- Another quick script showed that there were 5 rows in `V$SYSTEM_EVENT` that were NULL:
    
    ```sql
    SELECT count(*) 
    FROM sql_fulltext
    WHERE event IS NULL;
    
    COUNT(*)
    --------
           5
    ```

    WTH?
- Looking at the `EVENT` column, showed a huge load of crud and nothing much like a proper Oracle event. Some of the data were:
    
    ```text
    rwp err: No dash in error string
    r removing error %d is [ ][ ]
    rrupt, error stack is [ ][ ]
    down and process is starting up
    indicate must count [ ][ ]
    ...
    ```

    WTH? (The sequel!)
- The next stop - I did say I didn't have half a brain didn't I - was the alert log, _where I should have been looking in the first place_! Bingo!:
    
    ```text
    WARNING: Oracle executable binary mismatch detected.
    Binary of new process does not match binary which started instance.
    ```
    

And there we have it. Noons was correct in as much as the version of statspack in use - 11.2.0.3 EE - didn't match the running database binary which was 11.2.0.3 SE. Somehow, someone (no, not me - but thanks for asking!) had managed to start the database running on SE rather than EE.

I'm thinking that this could have been done when an SE environment was enabled in a session, and someone simply did an `export ORACLE_SID=whatever` not realising that EE was required. [This posting](/posts/2013/03/setting-oracle-environment-in-scripts/ "Setting Oracle Environment in Scripts") might help in that case! :-)

After shutting down the database, making sure that the correct environment was set, a restart of the database got rid of the messages in the alert log, and a snapshot was successfully executed.

So, an interesting challenge that could have been resolved earlier if I'd gone straight to the alert log rather than dicking about thinking I knew that it must have been the reinstall I did previously! That'll teach me to think then!

And by the way, I know (oops) from previous experience, that the snapshot code in `V$SQL` will have table names and commands in upper case, which is why I used upper case tests for `INSERT` and `SYSTEM_EVENT` in the script above. If I wasn't so sure, I'd have done this instead:

```sql
SELECT sql_fulltext
FROM   v$sql
WHERE  DBMS_LOB.INSTR(upper(sql_fulltext), 'SYSTEM_EVENT') <> 0
AND    DBMS_LOB.INSTR(upper(sql_fulltext), 'INSERT') <> 0;
```

Cheers.
