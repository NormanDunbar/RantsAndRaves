---
title: "Introduction to Oracle Datapump - Part 2"
date: "2013-09-05"
categories: 
  - "oracle"
---

In this, the second part of the Introduction to Oracle Datapump mini-series, we take a look at importing dump files using `impdp`. If you missed the first part which concentrated on exporting with expdp, [have a read of it here](/posts/2013/08/introduction-to-oracle-datapump-part-1/ "Introduction to Oracle Datapump – Part 1"). Once again, the following is a quick introduction for people like me - running Oracle on Linux and slightly averse to change! ;-)

## Introduction to Datapump Imports

> All of the following is based on 11.2.0.2 Enterprise Edition of Oracle, running on Suse Linux, Enterprise Edition version 11 SP 0.

> Please note. In the following, any Linux commands that need to be executed as the root user, are prefixed with a '#' prompt. All other commands are prefixed by a '$' prompt, and these should be executed as the oracle user.

Before we start, we need to make sure we have a couple of things (technical term) set up.

## Prerequisites

When you come to use Datapump's `impdp` utility, you need to have a pre-existing Oracle Directory within the database being imported in to. This directory object tells datapump - where to find dump files and where to write log files. As mentioned in [the previous article](/posts/2013/08/introduction-to-oracle-datapump-part-1/ "Introduction to Oracle Datapump – Part 1"), every database created has a directory already set up for datapump to use. It is called `DATA_PUMP_DIR` and defaults to the location $ORACLE_HOME/rdbms/log.

This isn't normally the best location, so you have a choice of amending (ie dropping and recreating) the current one, or creating a new one for our own use. I find it best to create a new one. I also like to set up a special datapump_admin user dedicated to carrying out exports and imports. Instructions on creating Oracle Directories and setting up the datapump_admin user, and its required privileges, were covered in the [the previous article](/posts/2013/08/introduction-to-oracle-datapump-part-1/ "Introduction to Oracle Datapump – Part 1"), and will not be repeated here.

## Importing

This article concentrates mainly on the importing of data from a database using the `impdp` utility which replaces the old `imp` utility we know and love so much!

before we start looking at specifics, be aware that when we Luddites use `imp` we need to be aware that it _appends_ data to existing tables (provided `ignore=y` of course) and if we didn't want this to happen, we either had to login and drop the tables in question, or at the very least, truncate them. With `impdp` we don't need to do this! We have two options to replicate the `ignore=y` parameter of `imp`, `CONTENT` and `TABLE_EXISTS_ACTION`.

The following sections describe each of these parameters in turn, and further down the page, there is a section describing what exactly happens when these parameters are used in certain combinations. **Beware**, some combinations produce misleading error messages - in 11gR2 at least.

### Content

The `content` parameter takes the following values:

- All - the default. `Impdp` attempts to create the objects it finds in the dump file, and load the data into them.

- Metadata_only - `impdp` will only attempt to create the objects it finds in the dump file. It will not attempt to bring in the data. This is equivalent to `rows=n` in `imp`.
    
- Data_only - `impdp` will not try to create the objects it finds in the dump file, but it will attempt to load the data. This is equivalent to `ignore=y` in `imp`.
    
    If there are objects in the dump file, which are not in the database/schemas/tablespaces being imported, then there will be errors listed to the log file, and screen, and those missing tables will _not_ be imported.

### Table_Exists_Action

This parameter tells `impdp` what to do when it encounters a table with or without data already in it. It takes the following values:

- Append - `impdp` will simply append the data from the dump file to the table(s) in question. Existing data will remain, untouched. This is the default option if `content=data_only`.

- Replace - `impdp` will drop the object, recreate it and then load the data. Useful, for example, if the definition of a table has been changed, this option will ensure that the new definition - in the dump file - is used. This value cannot be used if the `content=data_only` parameter is in use.

- Skip - the default, unless `content=data_only`. `Impdp` will not attempt to load the data and will skip this table and move on to the next one. This is very handy if an import went wrong, for example, and after sorting out the failing table, you simply restart the import and have it skip the tables it has already completed.

- Truncate - `impdp` will truncate any tables it finds, but not drop them, and then load the data.

### Legacy Mode Parameters

Of course, if you can't be bothered to learn the new parameters for `impdp`, then from 11gR2 onwards, you can specify a number of the old `imp` parameters and have `impdp` convert them to the new ones for you. It's best to learn the new ones though, you never know if Oracle will drop legacy mode at some future date.

Right, onwards, we have data to import!

### Import a Full Database Dump File

We created a full database export file - full.dmp - last time, which we can use to import back into the same, or a different database. As before, it is usually wise to put the required parameters in a parameter file, and specify that on the command line - this is especially true if you need to have double or single quotes in some of the parameters - as these will need escaping on the command line, but not in the parameter file.

```bash
$ cat fullimp.par
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
dumpfile=full.dmp 
logfile=full.imp.log 
**full=y**
content=all
table_exists_action=truncate
```

If you omit the password from the `userid` parameter, `impdp` will prompt you.

Running a full import is now a simple matter of:

```bash
$ impdp parfile=fullimp.par
```

What happens next is that a pile of "stuff" scrolls up the screen, some of which is useful, some not so. Straight away, you will notice (because I've highlighted a couple!) a number of errors about objects already existing. This is because we have `content=all`. Had we simply had `content=data_only` these errors would not have appeared.

Here is an excerpt with some of the good bits highlighted:

```text
Import: Release 11.2.0.2.0 - Production on Sat Aug 31 16:47:13 2013
Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.
Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production
With the Partitioning option

Master table "DATAPUMP_ADMIN"."SYS_IMPORT_FULL_01" successfully loaded/unloaded
**Starting "DATAPUMP_ADMIN"."SYS_IMPORT_FULL_01"**:  datapump_admin/******** parfile=fullimp.par
Processing object type DATABASE_EXPORT/TABLESPACE
**ORA-31684: Object type TABLESPACE:"SYSAUX" already exists
ORA-31684: Object type TABLESPACE:"UNDOTBS1" already exists**
...
Processing object type DATABASE_EXPORT/PASSWORD_VERIFY_FUNCTION
...
Processing object type DATABASE_EXPORT/SCHEMA/USER
**ORA-31684: Object type USER:"OUTLN" already exists
ORA-31684: Object type USER:"ORACLE" already exists
ORA-31684: Object type USER:"NORMAN" already exists**
...
Processing object type DATABASE_EXPORT/GRANT/SYSTEM_GRANT/PROC_SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/GRANT/SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/ROLE_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/DEFAULT_ROLE
...
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
**ORA-39120: Table "SYSTEM"."DEF$_DESTINATION" can't be truncated, data will be skipped. Failing error is:
ORA-02266: unique/primary keys in table referenced by enabled foreign keys**
...
. . imported "APP_OWNER"."TEST"                          5.031 KB       5 rows
. . imported "NORMAN"."NORM"                             5.023 KB       4 rows
...
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/POST_TABLE_ACTION
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TRIGGER
Processing object type DATABASE_EXPORT/SCHEMA/POST_SCHEMA/PROCACT_SCHEMA
Processing object type DATABASE_EXPORT/AUDIT
**Job "DATAPUMP_ADMIN"."SYS_IMPORT_FULL_01" completed with 700 error(s) at 16:50:18**
```

Well, that went well - but full imports - even with `imp` - usually do create masses of errors. They are probably best avoided. I suppose they _might_ be useful when you have created a brand new database, with only the Oracle required tablespaces etc, and you are happy to have `impdp` bring in a clone, almost, of another database. Maybe! ;-)

> You will notice that because of all the extra errors caused by numerous objects that already exist, that checking the log file for real errors will be a little difficult.
> 
> If you already know that _all_ the objects, present in the dump file, exist in the database that you are importing into, then use the `content=data_only` parameter which will prevent these errors from appearing. Remember to set a suitable value for the `table_exists_action` parameter as well, otherwise the default action is to skip whichever objects it finds already existing.
> 
> You will not be allowed to use `replace` because there will be no metadata in the import, so there will be no commands to recreate the objects. You only have `append`, `truncate` or `skip` available. If you try to use a forbidden option, you will see the following error:
> 
> ORA-39137: cannot specify a TABLE_EXISTS_ACTION of REPLACE for a job with no metadata
> 
> Running the above full import again, but with a data_only import this time, resulted in far fewer errors:
> 
> ...
> Job "DATAPUMP_ADMIN"."SYS_IMPORT_FULL_01" completed with 21 error(s) at 16:52:41
> 
> And this time, these were due to dropping tables that were in use as the parent of a foreign key constraint.

You will see highlighted above, the usual job name details. The format of the job names was described in the previous article in this mini-series, and will not be discussed further here.

Following the job details - and as already mentioned, a table of the same name will be created within the datapump_admin user while the job is running - we start to see the list of various objects that already exist. Then, eventually, in amongst all the errors, we see the tables actually being imported. Phew!

Note, on the final line, the number of errors I have to check to ensure that they are not critical. Hmmm, I really do not like database full imports, never have done and I doubt I ever will. I think we should move on, quickly, and take a look at schema imports instead.

### Importing Schemas

Importing a scheme or schemas is a better way to bring data into an existing database. You can, if you wish, import a particular schema from a full dump - you don't have to have exported the schemas specifically - the example below will demonstrate this.

As with `expdp`, you simply need to specify the `schemas=` parameter in your parameter file or on the command line:

```bash
$ cat schema_imp.par
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
dumpfile=full.dmp 
logfile=schema.imp.log 
**schemas=norman**
content=all
table_exists_action=replace
```

```bash
$ impdp parfile=schema_imp.par 
```
```text
Import: Release 11.2.0.2.0 - Production on Mon Sep 2 16:17:24 2013
Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.
Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production
With the Partitioning option
Master table "DATAPUMP_ADMIN"."SYS_IMPORT_SCHEMA_01" successfully loaded/unloaded
S**tarting "DATAPUMP_ADMIN"."SYS_IMPORT_SCHEMA_01"**:  datapump_admin/******** parfile=schema_imp.par 
Processing object type DATABASE_EXPORT/SCHEMA/USER
**ORA-31684: Object type USER:"NORMAN" already exists**
Processing object type DATABASE_EXPORT/SCHEMA/GRANT/SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/ROLE_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/DEFAULT_ROLE
Processing object type DATABASE_EXPORT/SCHEMA/PROCACT_SCHEMA
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
. . imported "NORMAN"."NORM"                             5.023 KB       4 rows
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/STATISTICS/TABLE_STATISTICS
**Job "DATAPUMP_ADMIN"."SYS_IMPORT_SCHEMA_01" completed with 1 error(s) at 16:17:35**
```

What happens if you specify the schema only dump file that was previously created, but supply the `full=y` parameter? Exactly as you would expect, the full dump file is imported and, in the following example, the _norman_ schema is once more imported - and existing objects are replaced.

```bash
$ cat schema2_imp.par 
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
**dumpfile=norman.dmp 
full=y**
logfile=schema.imp.log 
content=all
table_exists_action=replace
```

```bash
$ impdp parfile=schema2_imp.par 
```
```text
Import: Release 11.2.0.2.0 - Production on Mon Sep 2 16:24:39 2013
...
**Starting "DATAPUMP_ADMIN"."SYS_IMPORT_FULL_01"**:  datapump_admin/******** parfile=schema2_imp.par 
...
Processing object type SCHEMA_EXPORT/USER
ORA-31684: Object type USER:"NORMAN" already exists
...
. . imported "NORMAN"."NORM"                             5.023 KB       4 rows
...
Job "DATAPUMP_ADMIN"."SYS_IMPORT_FULL_01" completed with 1 error(s) at 16:24:49
```

So, the old `imp` behaviour is still available to use!

### Importing Tablespaces

As before, when you wish to import a tablespace, it can be from a full dump file, or a tablespace level dump file - provided that the tablespace(s) you want to import are actually present in the file. A table or schema level dump file will not be suitable.

To import at the tablespace level, add the `tablespaces=` parameter in your parameter file or on the command line:

What will happen if, for some reason, a tablespace - call it tools - exists in the dump file, but not in the database? Well, it will not be recreated - `impdp` will _not_ recreate the tablespaces, only the objects within them.

```bash
$ cat tablespace_imp.par 
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
**dumpfile=full.dmp** 
logfile=tablespace.imp.log 
**tablespaces=users**
content=all
table_exists_action=replace
```

```bash
$ impdp parfile=tablespace_imp.par 
```
```text
Import: Release 11.2.0.2.0 - Production on Mon Sep 2 20:05:19 2013
...
Master table "DATAPUMP_ADMIN"."SYS_IMPORT_TABLESPACE_01" successfully loaded/unloaded
**Starting "DATAPUMP_ADMIN"."SYS_IMPORT_TABLESPACE_01"**:  datapump_admin/******** parfile=tablespace_imp.par 
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
. . imported "APP_OWNER"."TEST"                          5.031 KB       5 rows
. . imported "NORMAN"."NORM"                             5.023 KB       4 rows
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/STATISTICS/TABLE_STATISTICS
Job "DATAPUMP_ADMIN"."SYS_IMPORT_TABLESPACE_01" successfully completed at 20:05:26
```

### Importing Tables

And finally, importing specific tables is as easy as the above. You should be completely at home with the parameter file and parameters by now:

```bash
$ cat tables_imp.par 
```
```text
userid=datapump_admin/secret 
directory=my_datapump_dir 
dumpfile=tables.dmp 
logfile=tables.imp.log 
**tables=norman.norm,app_owner.test**
content=all
table_exists_action=replace
```

```bash
$ impdp parfile=tables_imp.par
```
```text
Import: Release 11.2.0.2.0 - Production on Thu Sep 5 09:09:01 2013
Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.
Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production
With the Partitioning option
Master table "DATAPUMP_ADMIN"."SYS_IMPORT_TABLE_01" successfully loaded/unloaded
**Starting "DATAPUMP_ADMIN"."SYS_IMPORT_TABLE_01"**:  datapump_admin/******** parfile=tables_imp.par 
Processing object type TABLE_EXPORT/TABLE/TABLE
Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
. . imported "APP_OWNER"."TEST"                          5.031 KB       5 rows
. . imported "NORMAN"."NORM"                             5.023 KB       4 rows
Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
Job "DATAPUMP_ADMIN"."SYS_IMPORT_TABLE_01" successfully completed at 09:09:05
```

## Content and Table_Exists_Action Parameters

The following is a summary of the actions or errors you will see when various combinations of these two parameters are used together.

### Content = All

- If table_exists_action is **append**, then you will see error messages for any objects that currently exist in the database plus, existing data will be left untouched and the contents of the dump file will be _appended_ to those tables present in both the dump file and the current database.

- If table_exists_action is **truncate**, then then you will see error messages for any objects that currently exist in the database plus, any object that is being imported, which exists in the database, will be _truncated_ prior to the data from the dump file being imported.

- If table_exists_action is **Replace**, then then you will see error messages for any objects that currently exist but those objects will subsequently be _dropped_ and _recreated_ prior to the data being imported.

- If table_exists_action is **Skip**, then you will see error messages for any objects that currently exist in the database and nothing will be imported for those existing objects. Objects which exists in the dump file but not in the database, will be created. But not tablespaces.

### Content = Data_Only

- If table_exists_action is **append**, then provided that the definition of the objects in the dump file matches that in the database, data will be _appended_ and no error messages shown for existing objects. If an existing table has a different definition to that in the dump file, errors will be shown. Any objects in the dump file that do not exist in the database will report an error and will not be created.

- If table_exists_action is **truncate**, then no errors will be shown for existing objects and those tables that exists in both the database and dump file will first be _truncated_ prior to the data being imported. Any objects in the dump file that do not exist in the database will report an error and will not be created.

- If table_exists_action is **Replace**, then you will receive an error as this is an invalid parameter for this option for the content parameter. You cannot replace an object when the metadata, used to recreate it, is not being imported.

- If table_exists_action is **Skip**, then no data are imported for existing objects. Any objects in the dump file that do not exist in the database will report an error and will not be created.

### Content = Metadata_only

- If table_exists_action is **append**, then objects which do not exist in the database, but do in the dump file will be created. Tables which already exist in the database will not be affected in any way, however, the following _misleading_ message will be displayed:
    
    Table "USER"."TABLE_NAME" exists. Data will be appended to existing table but all dependent metadata will be skipped due to table_exists_action of append
    
    This is misleading as _no data will be loaded_ at all. We are specifying `contant=metadata_only` after all!

- If table_exists_action is **truncate**, then existing tables will be _truncated_, no new data will be loaded. New objects will be created from the dump file. Existing tables will cause the following message to be displayed:
    
    Table "USER"."TABLE_NAME" exists and has been truncated. Data will be loaded but all dependent metadata will be skipped due to table_exists_action of truncate
    
    This message is misleading as there will be _no data loaded_, the table will be empty after the import. If the dump file's metadata is different from the existing table definition, then the existing table will remain in force.

- If table_exists_action is **Replace**, then no messages will be displayed for existing tables. These will be _dropped_ and _recreated_ using the metadata from the dump file. No data will be loaded, so they will be empty after the import. New objects in the dump file will be created.

- If table_exists_action is **Skip**, then nothing will be done for existing objects. New objects, in the dump file, will be created. No data will be loaded. The following - correct - message will be produced for existing tables:
    
    Table "USER"."TABLE_NAME" exists. All dependent metadata and data will be skipped due to table_exists_action of skip.
    

## Importing - Cheat Sheet

The following is a list of "cheats" - basically, a list of the parameters you would find useful in doing a quick import at any level from full database right down to individual tables. As before, I have listed each one in the form of a parameter file - for ease of (your) copy and paste.

The following parameter files assume that you have set up a suitable Oracle Directory object within the database being exported, however, the first part of the cheat sheet summarises the commands required to create one, and a suitably endowed user to carry out the exports.

### Create an Oracle Directory

The following is executed in SQL*Plus, or similar, as a SYSDBA enabled user, or SYS:

```sql
create directory my_datapump_dir as '/your/required/location';
```

This location is where all the dump files need to be copied into before running an `impdp`. The log files for the imports will be created here as the jobs run.

The following must executed as root, unless the location is already owned by the oracle account of course!

```bash
mkdir -p /your/required/location
chown oracle:dba /your/required/location
```

### Create a Datapump User Account and Privileges

The following is executed in SQL*Plus, or similar, as a SYSDBA enabled user, or SYS:

```sql
create user datapump_admin identified by secret_password
default tablespace users 
quota unlimited on users;

grant create table to datapump_admin;
grant datapump_exp_full_database to datapump_admin;
grant datapump_imp_full_database to datapump_admin;
grant read, write on directory **my_datapump_dir** to datapump_admin;
```
### Full Import

This one is suitable for a database where the objects already exist. Tables will be trunccated prior to the data replacing the existing contents.

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=full.dmp 
logfile=full.imp.log 
**full=y**
content=data_only
table_exists_action=truncate
```

### Schema Import

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=full.dmp 
logfile=schema.imp.log 
**schemas=user_a,user_b**
content=all
table_exists_action=replace
```

### Tablespaces Import

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=full.dmp 
logfile=tablespace.imp.log 
**tablespaces=users**
content=all
table_exists_action=replace
```

### Table Import

```text
userid=datapump_admin/secret 
directory=**my_datapump_dir** 
dumpfile=full.dmp 
logfile=tables.imp.log 
**tables=[user.]table_a,[user.]table_b**
content=all
table_exists_action=replace
```

### Remapping Schemas and/or Tablespaces

In the old days, we could specify parameters such as `from_user` and `to_user` to change the resulting owner of the imported objects. We can do this too with `impdp` as follows.

#### Tablespaces

On the command line, or in the parameter file, simply add one of the following for each tablespace you wish to remap:

```text
remap_tablespace=export_tablespace:import_tablespace
```

Any objects in the `export_tablespace` will be mapped into the `import_tablespace` instead. If you have more than one tablespace to map from, then simply use more than one `remap_tablespace` parameter.

```text
remap_tablespace=old_users:users
remap_tablespace=old_data:archived_data
```

#### Schemas

On the command line, or in the parameter file, simply add one of the following for each schema you wish to remap:

```text
remap_schema=from_user:to_user
```

Any objects in the `from_user` will be mapped into the `to_user` instead. If you have more than one tablespace to map from, then simply use more than one `remap_schema` parameter.

```text
remap_schema=other_user_1:norman
remap_schema=other_user_2:norman
remap_schema=other_user_3:norman
```