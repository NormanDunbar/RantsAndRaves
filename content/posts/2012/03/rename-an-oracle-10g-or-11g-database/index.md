---
title: "Rename an Oracle 10g or 11g Database"
date: "2012-03-07"
categories: 
  - "oracle"
---

Due to a document not being supplied to me recently, I built a few 11g databases with the wrong SID. I needed to change them all. I found a few web pages, Oracle and others, on using the `nid` utility to do just that, however, they were incomplete. The full process is described here.

The following has been tested fully on an 11.2.0.2 Oracle database. The `nid` utility used also exists in Oracle 10.2.0.5 and possibly other versions that I do not have access to at the moment.

- Backup the database or ensure a backup exists in RMAN. This shouldn't be necessary but you are a DBA right? And chances we do not take!

- `create pfile='/home/oracle/initnew_sid.ora' from spfile;` this ensures that the pfile you are about to edit contains all the current settings. It is saved in a non-standard location for safety.

- Shutdown the database cleanly. Use `shutdown` or `shutdown immediate`.

- `Startup mount` the database.

- Check the `open_cursors` parameter. If you have a lot of data files you might need to increase this setting temporarily:
    
    `alter system set open\_cursors=1500 scope=memory;`
    

- In a shell session run the following `nid` command:
    
    `nid target=/ dbname=new\_sid setname=y logfile=new\_name.log`
    
    The logfile created will show details of what just happened and should be checked for errors. As far as I am aware, but don't trust me on this, errors will not change the database at all.
    
    The instance will also have been shut down at the end of the `nid` command.

- Edit `/etc/oratab` to change the old sid to the new one.

- Edit `tnsnames.ora` to do likewise. Also applies to OID, LDAP, whatever you use for alias resolution.

- Stop the appropriate listener, edit `listener.ora` and restart the listener. If the listener in question serves other databases, just edit the `listener.ora` file and run `lsnrctl reload listener_name`.

- Copy the newly created `/home/oracle/initnew_sid.ora` to `$ORACLE_HOME/dbs/initnew_sid.ora` then edit the new file and change the `db_name` parameter.

- You may, if desired, delete the old files `$ORACLE_HOME/dbs/initold_sid.ora` and `$ORACLE_HOME/dbs/spfileold_sid.ora` as they are no longer needed. (Unless you plan on renaming the database back again of course!)

- Export `ORACLE_SID` as the new sid. `ORACLE_HOME` will remain the same as before.

- Run the `orapwd` command to create a new password file if required.

- Startup the database.

- `create spfile='oracle_home/dbs/spfilenew_sid.ora' from pfile;` Obviously substituting the correct values for new\_sid and oracle\_home.

- Shutdown and restart the database to use the new spfile.

- Check it all just worked:
    
    `select name from v$database;`
    `show parameter db\_name`
    

That's the simple bit and in theory is all you need to do. However, be aware that any data files, control files, recovery file destinations etc still have their old format names. You may end up with a database called `fred` located in `/srv/barney/oradata` for example. If you wish to go down the route of renaming everything to suit the database name, see [part 2](/posts/2012/08/rename-an-oracle-10g-or-11g-database-part-2/ "Rename an Oracle 10g or 11g Database – Part 2") of this discussion, [here](/posts/2012/08/rename-an-oracle-10g-or-11g-database-part-2/ "Rename an Oracle 10g or 11g Database – Part 2").

As the control files are not overwritten by this process, and because the `DBID` doesn't change, all your `RMAN` backup information is safe and even after the database has been renamed, you can use an old backup to restore and recover the database. I've tested this on a tablespace recovery to be sure.

> Ok, when I say your backups are safe, they are, but if you try to restore the controlfile from an old pre-rename dump, it will restore with no errors. However, when you attempt to `mount` the database you will be informed that the name in the controlfile doesn't match the database. Edit the restored controlfile(s) to change the name and continue.

I'm pretty sure that because the `DBID` doesn't change, an `RMAN` catalogue will not lose any backup details either. On my test system, I don't yet have a catalogue to play with, so best you test on an _expendable_ database first. Just saying!

Online and backed up archived logs as well as the online redo logs are used quite happily to apply any required REDO.

It just works!
