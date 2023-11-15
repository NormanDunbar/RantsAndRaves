---
title: "RMAN Error ORA-15028: Archived Log Not Dropped."
date: "2014-09-16"
categories: 
  - "oracle"
---

The following error popped up in an RMAN backup which was attempting to delete archived logs that had been backed up twice, at least, and were created more than two days ago:

```text
RMAN-03009: failure of delete command on default channel at 09/12/2014 08:43:50
ORA-15028: ASM file '+FRA/MY\_DBNAME/archivelog/2014\_09\_09/thread\_1\_seq\_35804.3258.857840113' not dropped; currently being accessed
```

(Database names changed to protect the innocent, as usual.)

David Marcos has a blog entry from September 2010 on this very matter at [http://davidalejomarcos.wordpress.com/2010/09/07/unable-to-delete-archive-log-and-fra-is-filling/](http://davidalejomarcos.wordpress.com/2010/09/07/unable-to-delete-archive-log-and-fra-is-filling/) which suggests killing the various _arc_ processes for the database in question, one by one. Oracle will restart them and the database will stay up.

Now I don't know about you, but in my case, this was a production database and I have a certain trepidation about killing off background processes at random in the _hope_ that it will cure a fault. That sort of approach may work for Windows faults and problems, but this is a _real_ server, running under Unix. ;-)

In the `asmcmd` shell at Oracle version 11.2 there is the useful `lsof` command that will return details of who, or what, has a file opened, but in my case, the version was only 11.1, which doesn't have anything much in the way of useful commands!

I decided, in the absence of `lsof` to search the alert log for the archived log's filename to see if any of the arc processes recorded having had a problem with the file. They didn't, but I did find the following:

```text
LOGMINER: Begin mining logfile ... +FRA/MY\_DBNAME/archivelog/2014\_09\_09/thread\_1\_seq\_35804.3258.857840113
LOGMINER: Begin mining logfile ... +FRA/MY\_DBNAME/archivelog/2014\_09\_09/thread\_2\_seq\_37567.2138.857839917
LOGMINER: End mining logfile ... +FRA/MY\_DBNAME/archivelog/2014\_09\_09/thread\_2\_seq\_37567.2138.857839917
LOGMINER: Begin mining logfile ... +FRA/MY\_DBNAME/archivelog/2014\_09\_09/thread\_2\_seq\_37568.2810.857841217
```

From the above, it is plain to see that it is most unlikely that the arc processes would have been the culprits here. One of the other DBAs had started a log mining session for a few files, but had not ended the session for two of them.

Getting the DBA to run a quick `DBMS_LOGMNR.REMOVE_LOGFILE(...)` and/or a `DBMS_LOGMNR.END_LOGMNR` resolved the problem. Thankfully, the session running the log mining was still open, I am not sure what would happen if the session had already been closed, perhaps a full database restart would have been required - but as Wikipedia often says, "needs validation".
