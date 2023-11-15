---
title: "Impdp Hangs Importing Materialized Views"
date: "2014-03-13"
categories: 
  - "oracle"
---

A simple exercise to refresh a schema in a test database caused no end of problems when it hung at 99% complete. The last message on screen indicated that it was importing the Materialized Views (yes, with a 'Z'). After a long time, the DBA running the import killed it, cleaned out, and restarted the whole process. Exactly the same happened.

### Background

- The databases in question were both 11.2.0.3 Enterprise Edition.
- The materialized views were created and refreshed from a table in another database, utilising a database link.

### Investigation

While the impdp was running, breaking into the session and running the status command, repeatedly shows that the import was at 99% completion and so many bytes had been processed. Using the command `status=120` we could see that this was not moving on at all as time went by. (The above command runs the status command every 2 minutes.)

Checking the server for the processes doing the import, DW00, we were able to extract the SID for the process:

```sql
select sid, serial# from v$session s 
where paddr = (
    select addr from v$process
    where spid = &PID
);
```

Running the above in SQL*Plus, and entering the Unix process id of the DW00 process for the database, we were able to find the SID and SERIAL# for the hung process.

Looking in V$SESSION_WAIT for that process, we could see that it was counting up from around 128 seconds, and was waiting on an event named "SINGLE-TASK MESSAGE".

Googling around for this event seemed to indicated that it was mainly responsible for a process to create a synonym for a table on the other end of a database link, taking up to 10 minutes to fail. Not quite our problem, but we might as well check.

In the importing database, we could see the SQL used to create the materialized views and noted the fact that they were all created from data held in another table, on the far end of a database link. Hmmm, suspicious!

Checking DBA_DB_LINKS we made a note of the HOST column, and in a shell session, tried a `tnsping` - no response.

A quick edit to the tnsnames.ora file to add in the appropriate details for these "hosts" and suddenly, the impdp session completed with no errors. This is good, but what exactly was going on?

### What Impdp Does Down a DB Link

A test session was set up whereby a materialized view was created with a data source at the far end of a database link. This was refreshed, checked, and exported before being dropped.

The database at the far end of the link was set up with a trigger that fired "after logon" and if the user in question was being logged into, set event 10046 at level 12 - might as well get more data than we need!

When we re-ran the import of the materialized view, we could see in the generated trace file that Oracle was connecting to the database and parsing the SQL statement that was used to refresh the data for the materialized view. Note, it was never executed or fetched from, only parsed. Basically, Oracle was checking that the source of the data was correct enough to be used by the materialized view when refresh time came around, when we were creating the materialized view.

So, when you are doing this sort of thing in future, make sure that any database links that exists in the schema(s) owning the materialized views, or that are being imported into the schema, are going to be valid and usable at the time the materialized view itself is imported. If not, you will see this wait and your import will never get past the materialised view section.

This problem may well also rear its ugly head if you have tables, views, or PL/SQL code in packages etc that make use of database links.
