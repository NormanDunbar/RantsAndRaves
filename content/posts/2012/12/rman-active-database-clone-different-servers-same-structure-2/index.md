---
title: "RMAN Active Database Clone - Different Servers, Same Structure"
date: "2012-12-06"
categories: 
  - "linux"
  - "oracle"
---

This post is all about cloning an 11g database from one server to another using an RMAN active database clone. This is not being done for Standby Database purposes, only to duplicate an existing database onto another server.

The physical structure on both servers is the same, some path names have been changed.

## Source Structure

- Database SID: msmdppr
- /srv/msmdp/oradata/msmdppr/
- /srv/msmdp/flashback_area/msmdppr/

Everything hangs off `/srv/msmdp/oradata/msmdppr` and there are `data`, `index`, `temp`, `undo`, `redo`, `ctrl`, `dbs`, `diag` directories there.

The `redo` and `ctrl` directories are also to be found under the flashback_area for this database. A copy of the redo files are held there and there are some of the control files as well. This is a separate LUN from the `oradata` structure.

These are all part of a multi-spindle SAM, so it's not as bad as it looks having everything in the "same place".

## Destination Structure

- Database SID: msmpr
- /srv/msm/oradata/msmpr/_all as above_
- /srv/msmdp/flashback_area/msmpr/_all as above_

The destination structure will end up the same as the source, and only the names will change. Where the source has _/msmdp/_ or _/msmdppr/_ the destination will be _/msm/_ or _/msmpr/_

## Destination Preparation

- Build the desired directory structures for the new database. In this example, I did the following, as the oracle user:
    ```bash
    $ mkdir -p /srv/msm/oradata/msmpr/
    $ mkdir -p /srv/msm/flashback_area/msmpr/
    
    $ cd /srv/msm/oradata/msmpr/
    $ mkdir -p data index temp undo redo ctrl dbs changetracking
    $ mkdir -p diag/rdbms/msmpr/msmpr/adump
    
    $ cd /srv/msm/flashback_area/msmpr/
    $ mkdir redo ctrl
    ```

- Create a _temporary_ password file for the new database. Set the password to be the same as the source database. It won't work otherwise. Ask me how I know?

## Source Preparation

- Make sure when you start the source database that there are no warnings about any deprecated parameters in the spfile. If there are, check the alert log and remove them. If you don't, they will get transferred to the destination database and the clone will fail!

- Recreate a pfile from the spfile. You must have an up to date pfile. You can use a minimum pfile on the destination server, but I find it is better to be explicit. I've had problems.

- Copy the new pfile over to the destination server. You can put it straight into $ORACLE_HOME/dbs if you wish, or put it somewhere else, and create a symbolic link to $ORACLE_HOME/dbs. The latter is what I did. All individual database startup and password files live in the `/dbs/` directory and are linked from there.

## Further Preparation

The remainder of the work needs some parts to be done on the source server and others to be done on the destination server.

- On the destination server, edit the pfile that was created on the source server and copied over. You need to change all occurrences of the source database name to the destination one, and make sure that all your paths etc are correctly defined for the new database

- Now add the following two parameters to the pfile:
    
    - db_file_name_convert='/srv/msmdp/oradata/msmdppr/', '/srv/msm/oradata/msmpr/', '/srv/msm/flashback_area/msmpr/', '/srv/msm/flashback_area/msmpr/'
    - log_file_name_convert='/srv/msmdp/oradata/msmdppr/', '/srv/msm/oradata/msmpr/', '/srv/msm/flashback_area/msmpr/', '/srv/msm/flashback_area/msmpr/'
    
    They will convert the filenames to the destination format. **Beware**, I have found that these parameters only substiture at the start of a parameter, not where the parameter includes the text to be replaced.

- Startup the destination database in NOMOUNT mode.

- On the destination server, create a listener for this database and start it up.

- On the destination server, create a tnsnames.ora entry for the new database.

- On the destination server, test that you can connect to the destination database with `sqlplus sys/password@msmpr as sysdba`. If it works, carry on, otherwise, fix it.

- On the source server, create a tnsnames.ora entry for the destination database.

- On the source server, test the connection. You need to be able to connect to the database with `sqlplus sys/password@msmpr as sysdba`

- On the source server, if the database is running in NOARCHIVELOG mode, shut it down, then restart it in MOUNT mode. If the database is running in ARCHIVELOG mode, it can either be OPEN or MOUNTed.

- On the source server, set the environment for the source database (msmdppr) and start RMAN. You may need to specify `$ORACLE_HOME/bin/rman` to get the correct "rman" if you are on a Linux server. There are other rmans!

- In the RMAN session, run the following commands:
    ```sql
    connect target sys/password
    connect auxiliary sys/password@msmpr
    
    duplicate target database to msmpr
    from active database
    nofilenamecheck
    password file;
    ```
    then leave RMAN to do its thing.
    
    In my case that meant an overnight job as there was around 6.5Tb to clone. Luckily RMAN, when configured for performance, will skip the completely virgin blocks in a data file which makes things a little quicker.

The RMAN commands are saying to clone the source database and copy accross the password file from the source database to the destination one. If you have any users with SYSOPER or SYSDBA roles granted, their passwords will be in the password file, so you'll need it on the destination server as well.

When it finishes, you will find the password file and a new spfile for the destination database in $ORACLE_HOME/dbs. if necessary, move them to where you want them, and create symbolic links back to $ORACLE_HOME/dbs.

## When Things Go Wrong

Problems happen. I've had my share, especially with RMAN clones!

In the event that the clone exercise fails at some point, don't panic. Log out of RMAN on the source server, and go back in.

On the destination server, shut down the database, you can ABORT if necessary. Delete the spfile for the new database, if it still exists, then restart the destination database in NOMOUNT mode again.

Fix the problem, and start again from the `connect target ...` command in RMAN.

One thing I did notice, in my xxxx_file_name_convert parameters, I set it to convert '/msmdp/','/msm/','/msmdppr/','/msmpr/' and it ignored the second pair of substitutions. It seems as if it can cope with the first pair being part of the file names in question, but not the second pair, they remained unconverted and my data file copies failed because the /msmdppr/ directory wasn't to be found.

in the end I gave up on it completely, and just forced it to convert the entire thing.

Good luck, RMAN, when it's working is great. When you struggle with it, you struggle!
