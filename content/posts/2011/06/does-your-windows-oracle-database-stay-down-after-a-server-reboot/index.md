---
title: "Does Your Windows Oracle Database Stay Down After A Server Reboot?"
date: "2011-06-21"
categories: 
  - "oracle"
  - "windows"
---

After a server reboot, your (windows) database stays down, even though you have it set to come up automatically - why? The services have started, just not the database.

The following is a clue: 

```bash
sqlplus "/ as sysdba"  
```
```text
SQL*Plus: Release 10.2.0.3.0 - Production on Tue Jun 21 07:29:08 2011 
Copyright (c) 1982, 2006, Oracle. 
All Rights Reserved. 

ERROR: ORA-01031: insufficient privileges
```

You now have two options to check, and both must be confirmed:

- Is the `oracle` user a member of the `ora_dba` group?
- Is `sqlnet.ora` correctly configured?

To check the first, proceed as follows (on the database server!):

- Start->Programs->Administrative Tools->Computer Management.
- Double-click Local Users and Groups.
- Double-click Groups.
- Double-click ora_dba (on the right) if it exists. If it doesn't, it should - so have it added.
- The list of members of the ora_dba group should appear. Make sure that the oracle user is listed. If not, add it to the group.
- If you made changes, logout of the oracle user and back in again, then continue.

To correctly configure sqlnet.ora, make sure that the following line appears:

`SQLNET.AUTHENTICATION_SERVICES=(NTS)`

If it is not found, then the default is `(NONE)` which requires a username and password always be supplied when connecting as `sysdba` or `sysoper`.

If it is set to anything else, other than `(ALL)`, then it should be ok - although some of the other settings may not use `(NTS)` - you need to test on your system to be sure.

Now that you can `connect "/ as sysdba"`, you can check the `oradim.log` (`%oracle_home%\database\oradim.log`) to see if there are any of these errors corresponding to your server reboot times:

```text
oradim.exe -startup 
...
ORA-01017: invalid username/password; login denied
```

You may also find entries in the application pages of the event viewer with similar messages: 

```text
ORACLE. "CONNECT" DATABASE USER: "/" PRIVILEGE: NONE CLIENT_USER: oracle CLIENT_TERMINAL: STATUS: 1031
```

The alert log shows no errors whatsoever. It won't, you didn't get that far!

If the `oradim.log` shows no errors of the above kind, you may not have the database configured to autostart. So, delve into the registry. 

If the two following keys exist, and have the value _true_, then all should be well on the next restart. Otherwise add/edit them manually. Setting _both_ to _true_ ensures that when you start and stop the _services_ the database will also start and stop.

When the server is rebooted, the services are told to stop, so they will stop the database cleanly _before_ the server goes down. The reverse is true on a server startup, the services are told to start and they tell the database to come on up!

```text
HKLM/software/oracle/key_**ora_home_name**/ora_**oracle_sid**_autostart 
HKLM/software/oracle/key_**ora_home_name**/ora_**oracle_sid**_shutdown
```

If all of the above pan out, next time you restart the server, the database _will_ come up!

You can test without bouncing the server simply by shutting down and restarting the database service from control panel. On a restart of the service, the database should also be back up and running. If not, get into the `oradim.log` again, and fix the problem you find.

Cheers.
