---
title: "Oracle deadlocks - what happens?"
date: "2009-08-05"
categories: 
  - "oracle"
---

From time to time an Oracle based application will encounter a deadlock. This happens when two (or more) sessions are holding onto a resource and waiting for another one before it can relinquish the one(s) it holds.

There is quite a lot of misunderstanding about deadlocks and what happens within Oracle to relieve the situation. Hopefully this blog entry will help to sort it all out.

First of all, a brief demonstration.

**Background.**

```sql
CREATE TABLE DEADLOCK (A NUMBER);
INSERT INTO DEADLOCK VALUES (1);
INSERT INTO DEADLOCK VALUES (2);
COMMIT;
```

**Session One.**

```sql
DELETE FROM DEADLOCK WHERE A=1;
```

Nothing wrong so far, the row will be deleted, but not (yet) `COMMIT`ed.

**Session Two.**

```sql
DELETE FROM DEADLOCK WHERE A=2;
```

Again, nothing wrong, the row will be deleted, but not (yet) `COMMIT`ed.

**Session One.**

```sql
DELETE FROM DEADLOCK WHERE A=2;
```

This session will hang waiting for Session Two to commit (in which case the delete will fail) or rollback.

**Session Two.**

```sql
DELETE FROM DEADLOCK WHERE A=1;
```

This session will hang waiting for Session One to commit (in which case the delete will fail) or rollback. In addition, we now have a deadlock condition.

After a few seconds, Oracle will detect the deadlock and pick one of the sessions and 'rollback' the statement. This is where we see our first misunderstanding about deadlocks.

- Oracle **does not** kill the session.
- Oracle **does not** kill the transaction.
- Oracle **only** kills the **statement**.
- Oracle **does** rollback the failing statement, but Oracle **does not** rollback the entire transaction that the failing statement is part of. (_Correction by Mark Bobak.)_
- PMON (Process Monitor) **does not** clear out the locks.

It is the responsibility of the session that detects the "_ORA-00060 deadlock detected while waiting for resource_" error to trap and handle the error by issuing a rollback (or a commit) command. **Only** once this has been done will the other session be able to continue.

If, like me, you run a test of the above using two separate SQL\*Plus sessions (or [TOAD](http://www.quest.com/toad-for-oracle/ "Toad for Oracle") sessions, whatever you like) you will find that the session that detects the deadlock will return to a prompt and allow you to enter new commands. The other session **remains hung** until such time as the other session releases the locks it took out on the rows it had deleted.

This demonstration uses DELETEs but UPDATE will show similar results.

Cheers.
