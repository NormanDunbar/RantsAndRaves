---
title: "ENQ: TS - Contention"
date: "2018-08-17"
categories: 
  - "oracle"
---

Thanks to [http://www.dbaglobe.com/2010/08/drop-temporary-tablespace-hang-with-enq.html](http://www.dbaglobe.com/2010/08/drop-temporary-tablespace-hang-with-enq.html) it was a simple matter to resolve the above enqueue wait on an attempt to drop a previously default temporary tablespace.

The session causing the problem was a DBSNMP session being run by the OEM agent on the server. The following script, from the above blog, allowed me to identify the session and sort out getting it 'removed' to allow the drop to continue.

```sql
SELECT   
    se.username username,
    se.SID sid,   
    se.serial# serial#,
    se.status status,   
    --se.sql_hash_value,
    --se.prev_hash_value,  
    se.machine machine,
    su.TABLESPACE tablespace,  
    --su.segtype,
    --su.CONTENTS CONTENTS
FROM   
    v$session se,
    v$sort_usage su
WHERE   
    se.saddr=su.session_addr;
```

I've commented out a few of the columns that I'm not interested in at this point, but maybe another time....

```
USERNAME SID SERIAL# STATUS  MACHINE            TABLESPACE  
-------- --- ------- ------- ------------------ ----------
DBSNMP   595 8273    INACTIVE myserver.mydomain NORMS_TEMP  
```
  

The sid and serial# were then used to remove the session and allow the tablespace to be dropped.

The script above is a general purpose "who is using temp space at the moment" query, and has been added to my arsenal.
