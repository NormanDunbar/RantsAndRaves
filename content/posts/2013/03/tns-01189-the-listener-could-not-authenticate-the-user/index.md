---
title: "TNS-01189: The listener could not authenticate the user"
date: "2013-03-19"
categories: 
  - "oracle"
---

Ever see this error? I have, just today. An interesting one to debug. I got there in the end though.

The database is running on a two node VERITAS cluster. To protect the innocent, I shall refer to these as node_04 and node_05, for that is similar to their real names! The database is not RAC, it runs on one node or the other, but never both. There is one instance and one database.

The DNS and/or whatever they use to ensure that traffic goes to the correct node (I have a good grasp of the technicalities, haven't I?) uses a host name of sgxxxx where xxxx is the database name. Tnsnames.ora and listener.ora use this special hostname in the configuration for the database alias resolution, and the listener.

The database is running on node_04 right now, but can be forced to run on node_05 due to a failure or a manual fail over. The listener follows the database and runs on the same node - wherever that happens to be. It is not possible for the listener to run on node_04 with the database on node_05, and vice versa.

I was attempting to get a status from lsnrctl for the listener:

```bash
Node_04> lsnrctl status lsnr_xxxx
```
```text
LSNRCTL for Linux: Version 11.2.0.2.0 - Production on 19-MAR-2014 10:11:03

Copyright (c) 1991, 2010, Oracle. All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=sgxxxx)(PORT=1603)))
TNS-01189: The listener could not authenticate the user
```

It failed, as shown above. Interesting because it has been working "forever".

Running a tnsping on the database had no problems. It returned with a zero millisecond response time. Pretty quick. So we know the listener is up and running, and responding, we just can't get a status.

```bash
Node_04> tnsping xxxx
```
```text
TNS Ping Utility for Linux: Version 11.2.0.2.0 - Production on 19-MAR-2014 10:12:23

Copyright (c) 1997, 2010, Oracle. All rights reserved.

Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = sgxxxx)(PORT = 1603)(CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = xxxx)))
OK (0 msec)
```

Next step was to check the listener.ora file to see if there was a password set for the listener - there was not. Hmmm.

*Insert a delay here while various obvious stuff was checked, and found to be ok. Everything works unless we use the listener to access the database. At that point, the TNS error occurs again.*

Eventually, I wondered what the sgxxx "special" hostname resolved to:

```bash
Node_04> ssh oracle@sgxxxx
password: 

...

Node_05> exit
Node_04>
```

Did you notice? I did, straight away. The special host name resolves to the other node in the VERITAS cluster. It should be resolving to the same node as the database - node_04.

The solution is to get the network guys and gals to fix it. It's a cluster of two nodes and should work accordingly. Sgxxxx needs to resolve to the cluster node where the database is running. It wasn't.

Nasty, but fun to debug. :-)
