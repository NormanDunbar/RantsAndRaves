---
title: "Trace Collier - An Oracle Utility to Mine 10046 Trace Files"
date: "2016-02-26"
categories: 
  - "oracle"
---

Have you ever needed to trawl through an Oracle Trace file to extract the SQL statements executed and found a whole load of bind variables have been used, so you need to find the `BINDS` section, extract the values, and virtually paste them into the parsed SQL statement?

No? This utility isn't for you then.

> **Update to version 0.16 and you too can compile and run this useful utility on Windows. See the Readme for details.**

### Trace Collier

* * *

**As of 3rd March, after a Cease and Desist letter from a German lawyer, https://www.wuesthoff.de, acting on behalf of their client, Synaxus, this utility has a new name, Trace Collier. It appears that Synaxus has an unrelated software product with a very similar name to my old name, and have registered it as a trade mark.** 

**Their product can be found at https://www.synaxus.de/index.php/en/traceminer/tm-40 why not take a look?**

* * *

**Note:** Updated 2nd December 2016 for version 0.19.

Trace Collier, as it is _now_ known, is available for download as source code, from my [Git Hub repository](https://github.com/NormanDunbar/TraceCollier). Click on the `Download Zip` button to get hold of it, then simply `unzip` it, `cd` to the created folder, edit the `config.h` file to suit your system, and then execute `make` to build the `Trace Collier` utility.

The `README` files (either markdown or HTML) have all the details.

So, given a trace file - which must have binds (10046, level 4 or 12 etc), the output will look something like this:

```
Trace Collier: Version 0.12
Processing: Trace file /full/path/to/nfpdpr_ora_16153.trc

---------------------------------------------------------------------------------------------------------------------------------------
EXEC Line :        Cursor ID : PARSE Line : SQL Text with binds replaced                           
---------------------------------------------------------------------------------------------------------------------------------------
     7555 :                  :            : COMMIT

     7556 :  #140136345356328:       7477 : INSERT INTO "U_NFP"."NFP_LW_MEASURED" ("SPECIES_RUN_ID","LENGTH","WEIGHT","SCALE_PACKET","CHARACTERISTIC_ID","AGE_BAND_ID","PRE_FIRST_SPAWN_ID","POST_FIRST_SPAWN_ID","TOTAL_SEA_AGE_ID","NUMBER_MARKS","NALL_AGE","TRAP_NUMBER","LW_MEASURED_COMMENT") VALUES (664499,186,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL)
     7581 :                  :            : COMMIT

     7582 :  #140136345356328:       7568 : select 00016EF5.0015.0006 from dual
     7619 :                  :            : COMMIT
...
```

So far, it has handled all the trace files I've thrown at it, but if yours breaks or doesn't produce the correct results, give me a shout. My email is in the `README`.

There's an option to run `Trace Collier` in `--verbose` (or just `-v`) mode. Don't! You have been warned. However, if you do (and you _really_ shouldn't!) make sure to redirect `stderr` to a file which will get very _very_ _**very**_ big.
