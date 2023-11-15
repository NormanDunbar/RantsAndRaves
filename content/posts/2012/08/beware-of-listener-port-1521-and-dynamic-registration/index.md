---
title: "Beware of Listener Port 1521 and Dynamic Registration"
date: "2012-08-03"
categories: 
  - "oracle"
---

As you already know, an Oracle database's `PMON` process will register your database with a listener without you having to do anything about it. However ...

This will only happen if the listener in question is running on port 1521. And it doesn't have to be named `LISTENER` either -- as I mistakenly thought-- it only has to be port 1521.

If you have a listener running on port 1521, **and** you have databases configured to connect via different listeners on other ports (on the same server) then your databases will be grabbed by the 1521 listener!

The following shows the output from `Lsnrctl status` for a listener running on port 1525. You can see that it is not handling anything at all. (Names changed to protect the guilty):

```text
...
Listening Endpoints Summary...

(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=test_server)(PORT=1525)))
Services Summary...
Service "test5" has 1 instance(s).
  Instance "test5", status UNKNOWN, has 1 handler(s) for this service...
The command completed successfully
```

Meanwhile, the listener listening on port 1521 has indeed grabbed everything! And as the `tnsnmaes.ora` specifies port 1525 for the test5 database, we will not connect!

```text
...
Listening Endpoints Summary...

(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=test_server)(PORT=1521)))
Services Summary...
Service "test1" has 1 instance(s).
  Instance "test1", status READY, has 1 handler(s) for this service...
Service "test1XDB" has 1 instance(s).
  Instance "test1", status READY, has 1 handler(s) for this service...
Service "test2" has 1 instance(s).
  Instance "test2", status UNKNOWN, has 1 handler(s) for this service...
Service "test3.world" has 1 instance(s).
  Instance "test3", status READY, has 1 handler(s) for this service...
Service "test4" has 1 instance(s).
  Instance "test4", status READY, has 1 handler(s) for this service...
Service "test5" has 1 instance(s).
  Instance "test5", status READY, has 1 handler(s) for this service...
The command completed successfully
```

If you want to prevent this from happening, you can add the following one line to the `listener.ora` for the listener listening on port 1521:

```text
DYNAMIC_REGISTRATION_lsnr_name = off
```

That way, the listener is prevented from accepting dynamic registration requests from the other databases' `PMON` process.
