---
title: "Deprecated Parameter Warning on Database Startup"
date: "2011-11-21"
categories: 
  - "oracle"
---

You know the feeling, your 10g or 11g database displays a warning message about your use of deprecated parameters at startup, but it doesn't say which parameters are deprecated?

You could look through the manuals to find the list of all deprecated parameters then go hunting in your spfile/pfile for those and remove them, or, you could simply look in the `alert.log`.

```text
...
Starting up ORACLE RDBMS Version: 10.2.0.4.
...
Deprecated system parameters with specified values:
  log_archive_start
End of deprecated system parameter listing
PMON started with pid=2, OS id=11541
...
```

There are a few more at 11g:

```text
...
Starting up:
Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 ...
...
Deprecated system parameters with specified values:
  log_archive_start
  hash_join_enabled
  max_enabled_roles
  background_dump_dest
  user_dump_dest
End of deprecated system parameter listing
PMON started with pid=2, OS id=5292
...
```

These databases were converted from 9.2.0.8 to 10g and 11g respectively. So, now you know what the deprecated parameters are, go fix them! ;-)

Cheers.
