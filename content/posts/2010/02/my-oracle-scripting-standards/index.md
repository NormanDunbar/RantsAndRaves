---
title: "My Oracle Scripting Standards"
date: "2010-02-10"
categories: 
  - "oracle"
---

Subject to unannounced changes of course, but ...

- **All supplied scripts will be QA'd to ensure adherence to the following standards.**

Any script that fails on one or more of the following rules will not be applied to any system. Instead it will be returned to the vendor for correction.

- **All vendor or internally supplied scripts must be able to be run - without error - in SQL\*Plus.**

We need this as a standard because of all the wonderful GUI tools out there in vendor land, Toad etc are fine - if you can guarantee that the person running the script has the same tool.  SQL\*Plus should be available in all Oracle installations. I much prefer Toad myself, but this is how it is.

I have seen (and run!) scripts in the past where semi-colons were missing off the end of statements. These had been tested and proved to work under some GUI or other that allowed this sort of thing - but come the day of reckoning, they failed in SQL\*Plus.

- **All scripts must spool to <scriptname>.log.**

This is so that we can see what errors occurred rather than hoping to spot them all as they zoom off the top of the screen at high speed!

- **The use of WHENEVER is not permitted.**

It's a pain when a script blows up at the first hurdle and then carries on, however, scripts which blow up and immediately vanish are even worse.

- **All scripts will explicitly set linesize, pagesize, trimspool etc.**

No assumptions about how the client is set up shall be made. The vendor must explicitly specify all required options.

- **Commit and/or rollback is not permitted in any script.**

Obvious really. If the vendor tests on a 6 row table and it all works, that's fine. What if I have a 3.6 billion row table (and I do!) that, if updated, could blow away my UNDO tablespace? I don't want some developer telling me it's ok to commit in such an event, I'm the DBA so I decide.

Scripts must instead prompt the DBA to commit or rollback as appropriate after checking the log file (see above) as required.

- **Mixing DDL and DML is forbidden.**

It is surprising how many vendors don't know how Oracle works. The simply do not realise that DDL implicitly commits any outstanding transactions before it starts to CREATE or ALTER or DROP or whatever. It then commits again when done (if successful).

If you do see scripts from vendors that mix and match DML and DDL (and invariable compound the error by having a commit at the end!) then you should be on your guard for other problems - they don't know what they are doing!

If it is impossible (!) to separate DML and DDL then it is imperative that all the DDL is executed first and all the DML executed last. This prevents the implicit commits from affecting the DML statements. However, it is advised that separate scripts are supplied for the DDL and DML parts.

- **Scripts that create stored procedure code must SHOW ERRORS.**

It's always nice when that well tested piece of code that creates a new package or whatever, fails to compile on my databases, but it doesn't tell me why. Slipping a SHOW ERRORS in at the end of the CREATE OR REPLACE ... is a nice little touch and keeps the DBA happy. The DBA is especially happy as the errors are logged in the spool file (see above) and can be sent straight back to the vendor for correction!

- **Do not drop temporary tables in the same script that created/used them.**

This shouldn't ever occur based on not mixing DDL and DML, but I have seen scripts which:

1. Create temp\_table as select \* from live\_table;
2. DML the temp table to massage the data.
3. Drop live\_table;
4. Create new live\_table as select \* from temp\_table;
5. Drop temp\_table;

Now, the obvious problem is this, it _always_ fails at step 1. Steps 2 through 4 obviously also fail resulting in the loss of the live table and all it's data. (10g recycle bin not withstanding!) and finally, we drop the temp table at step 5 losing all the data completely.

At this point, it's back to the backups - you did take a backup didn't you?
