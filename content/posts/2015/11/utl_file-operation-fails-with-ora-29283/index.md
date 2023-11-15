---
title: "UTL_FILE Operation fails with ORA-29283"
date: "2015-11-25"
categories: 
  - "oracle"
---

A process that called ''UTL_FILE'' was failing in the test system, but worked fine with _exactly_ the same set up in production. Why? The error was ORA-29283: invalid file operation. How do we find out _exactly_ why it was failing?

MY_DIRECTORY is a directory, owned by SYS with READ and WRITE privileges granted to a schema that uses it to create, write and read files in that location.

The oracle account on the server can create and read files in the directory location, `touch` and `cat` prove this.

Running a PL/SQL package, however, fails. The failing code was reduced to the following test sample:

```sql
declare
 v_fd utl_file.file_type;
begin
 v_fd:=utl_file.fopen('MY_DIRECTORY','norman.txt','w');
 utl_file.fclose(v_fd);
end;
/
```

Which blows up with the less than helpful message:

```text
ERROR at line 1:
ORA-29283: invalid file operation
ORA-06512: at "SYS.UTL_FILE", line 536
ORA-29283: invalid file operation
ORA-06512: at line 4
```

Here's a nice trick, stolen blatantly from Michael Schwalm at http://blog.dbi-services.com/troubleshooting-ora-29283-when-oracle-is-member-of-a-group-with-readwrite-privileges/ which shows how to actually see what the real underlying problem is for this exception.

```sql
SQL> -- Change define, we need to use an ampersand.
SQL> set define #

SQL> -- Get my current process ID into a variable.
SQL> column spid new_value unix_pid
SQL> select spid from v$process p 
  2  join v$session s on p.addr=s.paddr 
  3  and s.sid=sys_context('userenv','sid');

SPID
------------------------
121080

SQL> -- Trace open calls from my session. 
SQL> -- Without the &, the host call never returns!
SQL> -- We know that it is the utl_file.fopen call that is 
SQL> -- failing, so only trace open calls.
SQL> host strace -e trace=open -p #unix_pid & echo $! > tmp.pid
Process 121080 attached - interrupt to quit

SQL> -- Paste in the offending code...
SQL> declare
  2    v_fd utl_file.file_type;
  3  begin
  4    v_fd:=utl_file.fopen('MY_DIRECTORY','norman.txt','w');
  5    utl_file.fclose(v_fd);
  6 end;
  7 /
```

That throws up the following _helpful_ message, followed closely by the expected Oracle exception message again (not shown):

```text
open(" /logfiles/MYDB/norman.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666) = -1 ENOENT (No such file or directory)
```

At this point, I need to press CTRL-C to detach the strace session.

```sql
SQL> ^C
Process 121080 detached
```

Looking at the above message, I can see (almost) straight away that the file path has a leading space. This implies that whoever set up the original directory, created it with a minor typo that is hard to detect when looking at DBA_DIRECTORIES.

The fix was simple:

```sql
SQL> create or replace directory MY_DIRECTORY as '/logfiles/MYDB';
SQL> grant  read, write on directory MY_DIRECTORY to [whoever needs it];
```

And now, past in the offending code again, and it "just works":

```sql
SQL> declare
  2    v_fd utl_file.file_type;
  3  begin
  4    v_fd:=utl_file.fopen('MY_DIRECTORY','norman.txt','w');
  5    utl_file.fclose(v_fd);
  6 end;
  7 /

PL/SQL procedure successfully completed.
```

