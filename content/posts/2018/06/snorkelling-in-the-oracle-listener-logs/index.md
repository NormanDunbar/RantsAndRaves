---
title: "Snorkelling in the Oracle Listener Logs."
date: "2018-06-14"
categories: 
  - "oracle"
---

(Snorkelling is not quite as in depth as a "deep dive"!)

Attempting to parse a `listener.log` will probably bend your brain, but I needed to do it recently to determine which unique servers and/or desktops and/or application servers were still connecting to a database prior to that database going down for maintenance. This was an exercise in confirming that the documentation we have, is correct.

According to the _[Net Services Administrator's Guide](https://docs.oracle.com/cd/E11882_01/network.112/e41945/trouble.htm#NETAG016)_, there are a number of different message types that can appear in a listener.log:

- A client connection request.
- A `RELOAD`, `START`, `STOP`, `STATUS` or `SERVICES` command, issued by `lsnrctl`.

However, I've found that a `tnsping` also logs a message - probably because it's a connection request, sort of, plus, regular "service update" messages also appear, and finally, error messages.

Each entry in the file consists of _up to_ 6 different fields, as follows:

- Timestamp.
- Connect Data.
- Protocol Information (optional).
- Event.
- SID or SERVICE (optional).
- Result code.

The fields are, as mentioned, separated by asterisks. It's very nice of Oracle to mention this in the documentation, but, actually scanning the file has shown that there can be more, or less! More on that later.

> In the following, servers, databases and IP addresses have been obfuscated to protect the innocent, me! Even the dates and times are somewhat fictitious.

#### Connection Requests

Connection requests come in two types, successful and failed. Both have, as far as my listener logs are concerned, the full complement of 6 fields:

```
15-MAY-2018 10:34:44 * (CONNECT_DATA=(CID=(PROGRAM=)(HOST=__jdbc__)(USER=))(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=35405)) * establish * ORCL_RW * 0
```

A failed connection request is normally followed by an error message.

```
14-JUN-2018 10:07:37 * (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=__jdbc__)(USER=root))(SERVICE_NAME=ORCL_RO)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=34082)) * establish * ORCL_RO * 12514
TNS-12514: TNS:listener does not currently know of service requested in connect descriptor
```

#### Lsnrctl Commands

These have a reduced complement of fields, only 4.

```
14-JUN-2018 10:14:34 * (CONNECT_DATA=(CID=(PROGRAM=)(HOST=my_server)(USER=oracle))(COMMAND=status)(ARGUMENTS=64)(SERVICE=LISTENER)(VERSION=202375680)) * status * 0
```

#### Tnsping Requests

These have 3 fields, separated by asterisks.

```
15-MAY-2018 10:36:43 * ping * 0
```

#### Service Updates

These also have 3 fields, separated by asterisks.

```
15-MAY-2018 10:34:50 * service_update * pnet01p1 * 0
```

#### Reload Requests

These have 4 fields, and the following is copied directly from the docs as I'm not allowed to carry out a reload on my listeners!

```
14-MAY-2009 00:29:54 * (connect_data=(cid=(program=)(host=sales-server)(user=jdoe))(command=reload) (arguments=64)(service=listener)(version=135290880)) * reload * 0
```

#### Service Registration

These have 3 fields, and the following is copied directly from the docs as all my databases registered before the start of the listener log!

```
14-MAY-2009 15:28:43 * service_register * sales * 0
```

#### Service Died

These have 3 fields, and the following is copied directly from the docs as none of my services have died, yet!

```
14-MAY-2009 15:51:26 * service_died * sales * 12537
```

#### Error Messages

These normally follow on from a failed connection request, or command, and only have a single field:

```
TNS-12514: TNS:listener does not currently know of service requested in connect descriptor
```

#### Timestamp Messages

It appears also that there can be numerous Timestamp lines in the listener.log, these too consist of a single field.

```
Fri Jun 01 14:15:30 2018
```

## Parsing the Listener Log

Enough background, moving on...

I'm using `awk` to process the log files, it works, I can tell it to use an asterisk as the field separator and so on. I'm not even using any of the special GNU `gawk` extensions as I don't have `gawk` on this server.

In this exercise, I'm really only interested in connection attempts, and they have (or should have) 6 fields separated by '*' characters. However, being a suspicious type, I better check:

```awk
awk -F* '{print NF;}' /u01/listener.full.log | sort -n -u

0
1
3
4
6
8
10
```

Hmm, looks slightly unpromising. Lets extract all those different line types to separate files,. If your `listener.log` is as big as mine, that exercise took a while! Don't bother with the zeros, they are the blank lines.

```bash
for x in 1 3 4 6 8 10
do
    awk -F* -v XXX=${x} '{if (NF == XXX){print $0;}}' /u01/listener.full.log > /u01/listener.${x}.log
done
```

```bash
ls -l /u01/listener*

  65013791 Jun 14 11:06 /u01/listener.1.log
    739825 Jun 14 11:07 /u01/listener.3.log
  45683981 Jun 14 11:07 /u01/listener.4.log
2288202297 Jun 14 11:08 /u01/listener.6.log
       342 Jun 14 11:08 /u01/listener.8.log
       382 Jun 14 11:08 /u01/listener.10.log
2399640823 Jun 14 10:23 /u01/listener.full.log
```

The listing above is slightly edited for ordering and space purposes. It shows just the size in bytes, the date/time and the file name.

- `/u01/listener.1.log` is the full list of error messages and timestamp records. Nothing to see here!
- `/u01/listener.3.log` is the list of `tnsping` type messages, version messages etc. Nothing to see here either!
- `/u01/listener.4.log` is a list of "service update" events, plus the `lsnrctl status` or `lsnrctl services` commands.
- `/u01/listener.6.log` is a list of the connection requests, successful or failed. This is what I'm interested in.
- `/u01/listener.8.log` is a list of corruptions, possibly caused by the listener being briefly unavailable. The format of this file seems to be two entries amalgamated into one. Best avoided!
- `/u01/listener.10.log` is a similar problem to `listener.8.log` above. Another set of corruptions.

In order to whittle down the amount of data I 'm scanning, I have decided to extract only those rows with 6 fields, and which have a zero response code. These are the successful connection attempts. This is what I'm trying to gather figures for.

I will use the following script to extract the data from my own `listener.6.log` file, which is the extract of the 6 field records from the full listener log. However, the script still checks - in case I wish to use it on the `listener.log` itself.

I'm only keeping the Connect Data, Protocol Data and the SID/Service as I don't need the rest.

```awk
#! /usr/bin/awk -f
#
# Parses the listener.log file (or whatever comes in on stdin) to
# find any line with 6 fields. These will be:
#
# $1 Date and time. Unwanted.
# $2 Connect Data.
# $3 Address data including host from where the request came from.
# $4 Usually "establish". Unwanted.
# $5 The service name connecting to. Kept, just in case.
# $6 The response code. 0 is good. We only want zero.
#
# The output will be only those fields we want from any connection request that worked.
#
#
# USAGE:
#
# cat listener.log | awk -f extract_listener.awk > listener.temp.log
#
# OR:
#
# awk -f extract_listener.awk < listener.log > listener.temp.log
#
# OR:
#
# ./extract_listener.awk < listener.log > listener.temp.log
#
# Norman Dunbar
# 07/04/2018.
#

# This happens before the start of the file.
BEGIN{
    # Set  the incoming field separator.
    FS="*";

    # Set  the output fields separator too.
    OFS="*";
}

# This happens for every record in the file.
{
    # Only interested in records with 6 fields...
    if (NF == 6) {
        # And of those, only successful connection requests.
        if ($6 == "0") {
            print $2, $3, $5;
        }
    }
}
```

I'm running this as follows:

```bash
./extract_listener.awk < /u01/listener.6.log > /u01/listener.temp.log
```

The output file looks vaguely like this:

```
 (CONNECT_DATA=(CID=(PROGRAM=)(HOST=__jdbc__)(USER=))(SERVER=DEDICATED)(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=43511)) * ORCL_RW
 (CONNECT_DATA=(CID=(PROGRAM=)(HOST=__jdbc__)(USER=))(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=47111)) * ORCL_RW
 (CONNECT_DATA=(CID=(PROGRAM=)(HOST=__jdbc__)(USER=))(SERVER=DEDICATED)(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=48619)) * ORCL_RW
 (CONNECT_DATA=(CID=(PROGRAM=JDBC Thin Client)(HOST=__jdbc__)(USER=norman))(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=40722)) * ORCL_RW
 (CONNECT_DATA=(CID=(PROGRAM=)(HOST=__jdbc__)(USER=))(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=47113)) * ORCL_RW
 (CONNECT_DATA=(CID=(PROGRAM=)(HOST=__jdbc__)(USER=))(SERVICE_NAME=ORCL_RW)) * (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.1.20)(PORT=47112)) * ORCL_RW
```

What I'm after is a list of unique programs, hosts and users from the connect data field, plus the host they came from on the protocol information field. I tried a couple of different script methods before finding a reasonably good workable one. The processing is:

- Use the '(' as a field separator for the input file.
- Use the '=' as a field separator for the output.
- Look for the text "CID=" in all fields.
- If "CID=" is found, then the desired program is in the next field, host in the one after and user in the one after that. The host (from the protocol data) is in the second last field.

Here's the script, which has the extremely meaningful name of `a.awk`!

```awk
#! /usr/bin/awk -f
#
# Parses whatever comes in on stdin (from extract_listener.awk) to extract 
# the PROGRAM, HOST and USER from the CONNECT DATA plus the HOST from the PROTOCOL
# DATA of a listener log file.
#
# We need to find "CID=" in any field, then output the following three fields -
# PROGRAM, HOST and USER, plus the second to last field - HOST.
#
# The input file fields are separated by '(' which will need double escapes.
# The output fields will be separated by '='.
#
# It appears that on AIX, printing "expression fields" doesn't work. :-(
#
# USAGE:
#
# cat listener.temp.log | awk -f a.awk > listener.a.log
#
# OR:
#
# awk -f a.awk < listener.temp.log > listener.a.log
#
# OR:
#
# ./a.awk < listener.temp.log > listener.a.log
#
# Norman Dunbar
# 07/04/2018.
#
BEGIN {
   FS="\\\\(";
   OFS="=";
}

{
    # Count fields minus 1.
    myNF = NF - 1;

    # Find CID= ...
    for (x = 1; x < NF; x++) {
       if ($x == "CID=") break;
    }

    if (x != NF) {
        program = x+1;
        host = x+2;
        user=x+3;
        print $program, $host, $user, $myNF;
    } else {
      print "CID= Not Found\\n";
      print $0;
    }
}
```

I did try to `print $x+1, $x+2, $x+3, $NF-1` but all I got was those digits! I'm using an AIX version of `awk` so that might account for my difficulties! The output looks like this:

```
PROGRAM=)=HOST=__jdbc__)=USER=))=HOST=172.20.238.35)
PROGRAM=JDBC Thin Client)=HOST=__jdbc__)=USER=norman))=HOST=192.168.1.20)
PROGRAM=JDBC Thin Client)=HOST=__jdbc__)=USER=briansmith))=HOST=192.168.10.40)
PROGRAM=JDBC Thin Client)=HOST=__jdbc__)=USER=webonline))=HOST=192.168.10.62)
```

Next, I need to process this and extract a csv file consisting on just the 4 data items I need - program, host, user and from host. The excellently named `b.awk` does this for me.

```awk
#! /usr/bin/awk -f
#
# Parses whatever comes in on stdin (from a.awk) to convert the input
# PROGRAM, HOST, USER and HOST (another host) into a CSV file.
#
# The input file fields are separated by '='.
# The output fields will be separated by ','. (It's CSV after all!)
#
# USAGE:
#
# cat listener.a.log | awk -f b.awk > listener.b.csv
#
# OR:
#
# awk -f b.awk < listener.a.log > listener.b.csv
#
# OR:
#
# ./b.awk < listener.a.log > listener.b.csv
#
# Norman Dunbar
# 07/04/2018.
#

BEGIN {
   FS="=";
   OFS=",";
}

# We get $2 = PROGRAM + ')'
#        $4 = HOST + ')'
#        $6 = USER + '))'
#        $8 = Calling Host + ')'
#
# Need to slice and dice. Oh, wrap the program in quotes in case "+ASM" comes up
# as this foxes Excel when imported! It thinks it's a formula because of the leading '+'.

{
   print "\\"" substr($2,1,length($2) - 1) "\\"",
         substr($4, 1, length($4) - 1),
         substr($6, 1, length($6) - 2),
         substr($8, 1, length($8) - 1);
}
```

That gives me this:

```
"",__jdbc__,,172.20.238.35
"JDBC Thin Client",__jdbc__,norman,192.168.1.20
"JDBC Thin Client",__jdbc__,briansmith,192.168.10.40
"JDBC Thin Client",__jdbc__,webonline,192.168.10.62
```

Now all I need to do is replace the empty programs with "Unknown Program" and any empty users with "Unknown User" and sort into a unique output, so here's the `sed` file I'm using to make the edits. It goes by the name of `sed.file` (I do like meaningful names!):

```
s/^"",/"Unknown Program",/
s/,,/,Unknown User,/
s/))) //
```

The last line is for those weird entries which have a very strange username! This minor edit gives me the following:

```
"Unknown Program",__jdbc__,Unknown User,172.20.238.35
"JDBC Thin Client",__jdbc__,norman,192.168.1.20
"JDBC Thin Client",__jdbc__,briansmith,192.168.10.40
"JDBC Thin Client",__jdbc__,webonline,192.168.10.62
```

So, finally, I can string it all together and sort the output into unique records, add a few headings and we are done:

```bash
./extract_listener.awk < listener.log |\\
./a.awk |\\
./b.awk |\\
sed -f sed.file |\\
sort -u > temp.csv
```

I have a `headings` file as follows:

```
PROGRAM,HOST,USERNAME,CALLING_HOST
```

Which I prepend to the above output, to get the final, desired csv file:

cat headings temp.csv > listener.csv

I now have the required listing of all the different programs in use to connect to the database, who connected and from which server. It makes for interesting reading, and surprisingly, matches the documentation!

#### Summary

There's lots of _interesting_ information in the listener log files, some of it useful, some of it not so useful. `Awk` is a pretty decent way of mining the file for the information you need, and, even better, you _don't_ have to write `perl` scripts either! At least with `awk`, you can read it again after 6 months, and still understand it.

Enjoy.
