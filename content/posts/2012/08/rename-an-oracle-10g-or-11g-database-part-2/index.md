---
title: "Rename an Oracle 10g or 11g Database - Part 2"
date: "2012-08-09"
categories: 
  - "oracle"
---

So, you renamed your database using the `nid` utility as outlined [here](/posts/2012/03/rename-an-oracle-10g-or-11g-database/ "Rename an Oracle 10g or 11g Database") but now you need (or want) to change all the file system names to suit. Read on.

In the following example, we have two mount points for the database. Files on these mounts are spread all over a pile of separate discs making up the LUN - so it's not as bad as it looks!

The two mounts are wrongly named at the moment since we changed the database name using `nid` and we would like to tidy things up. The current names are:

- `/srv/old_name/oradata/old_sid/`
- `/srv/old_name/flashback_area/old_sid`

What we would like to see after the renaming is the following:

- `/srv/new_name/oradata/new_sid/`
- `/srv/new_name/flashback_area/new_sid`

First of all, you need a current pfile. If you don't have one already (from the rename exercise) then create one as follows:

```sql
create pfile='/home/oracle/initnew_sid.ora' from spfile;
```

We also need a control file trace taking, so we do this next:

```sql
alter database backup controlfile to trace as '/home/oracle/new_sid_controlfile.sql';
```

{{< alert theme="info" >}}
It should be obvious that you wouldn't want the pfile and control file trace to be located on the file system that is about to be moved. If they are, you will have to wait for the Sys Admins to complete the remounting exercise before you can do your own editing in preparation for restarting the database.
{{< /alert >}}

The next thing to do is backup the database. Shut it down and take a cold backup in your preferred manner - `RMAN` or some other utility. Better safe than sorry after all.

In a shell session, as the oracle user, remove (or move) the `initnew_sid` and `spfilenew_sid.ora` files, if present, from $ORACLE\_HOME/dbs:

```bash
rm $ORACLE_HOME/dbs/initnew_sid.ora` `rm $ORACLE_HOME/dbs/spfileold_sid.ora
```

Edit the newly generated parameter file as required and move it from `/home/oracle/initnew_sid.ora` to `$ORACLE_HOME/dbs/initnew_sid.ora`. Things to change will include some or all of the following:

- `audit_file_dest`
- `control_files`
- `db_recovery_file_dest`
- `diagnostic_dest`
- `log_archive_dest`
- `etc`

Anything that has been moved (or will be moved) by renaming the mount points will require changing.

Next, the control file trace that was created needs to be edited. There is much waffle to be removed and a choice of script to generate. Edit the file `/home/oracle/new_sid_controlfile.sql` and:

- Find the section of the trace file that is applicable to your wish to start the database with `RESETLOGS` or not. For `NORESETLOGS` it is the first section, near the top of the trace, otherwise it is the second section near the bottom.
    
    For my own databases, I have chosen not to reset the logs, so I use the top section of the trace file.
- Delete all the comment lines above the section you want. You need to start the script from the `STARTUP NOMOUNT` command in the appropriate section.
- Scroll down, but retain all the lines from (and including) `STARTUP NOMOUNT` until you find a comment line that looks something like `-- End of tempfile additions`. Delete that line, and everything else to the end of file.

You should now be in possession of a script that begins with `STARTUP NOMOUNT` and ends with a few lines adding your `TEMPFILE`s back into your temporary tablespace9s).

- Run a global search and replace to ensure that all filenames match with the new locations.
- If the database is in ARCHIVELOG mode, pay particular attention to the commented out commands to `ALTER DATABASE REGISTER LOGFILE` and make sure you enter the full path to an existing log file. You can ignore this part if your database is running in NOARCHIVELOG mode.
- Save the file.

When the Sys Admins have given you back the file systems mounted on their new locations, it's a simple matter to:

```sql
connect / as sysdba
@/home/oracle/new_sid_controlfile.sql
```


When that has finished, everything should now be in order. You may create a new spfile in the usual manner:

```sql
create spfile='/home/oracle/spfilenew_sid.ora' from pfile; 
shutdown 
...
host cp /home/oracle/spfilenew_sid.ora $ORACLE_HOME/dbs/ 
startup
...
```

Nothing to it!
