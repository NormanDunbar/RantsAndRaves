---
title: "Cannot Send Emails, or Read Web Servers From Oracle 11g"
date: "2013-02-05"
categories: 
  - "oracle"
---

Accessing a web server or an email server, directly from within a database, used to be quite simple. However, it all stops working at 11g. Why is that and what can be done to fix it?

## Introduction

Prior to Oracle 11g, any user in the database wishing to use the various network packages - UTL_HTTP, UTL_SMTP, UTL_TCP, UTL_MAIL etc, and their predecessors, only requires to be granted EXECUTE privileges on the appropriate package(s).

From 11g onwards this is no longer the case. Execute privilege is still required, however, further fine grained access control has been added to the database to restrict the networks reachable, port ranges allowed, and even down to the actual start and end dates and times that the access will be allowed.

This fine grained control is achieved via ACLs. (Access Control Lists.) and the XML Database (XDB) product which must be installed on the database.

## Information Requirements

If an application wishes to use network resources, then the following information must be obtained, in advance, of the application being set up in the database. It is advised that it be documented in the database and/or application documentation

### Email

For an application connecting directly to a mail server, not via sendmail, or equivalent:

- Hostname of the email server, or IP address of same.
- Port number(s) in use for the email server.
- Database username (ie schema names) of all schemas that will be executing the code that sends emails.
- If required, a range of dates and times when the service should be accessible.

### HTTP

For an application connecting directly to a web server or URL:

- Hostname of the web server, or IP address of same.
- Port number(s) in use for the email server.
- Database username (ie schema names) of all schemas that will be executing the code that connects to the web or email server.
- If required, a range of dates and times when the service must be accessible.

## Setting up ACLs

The SYSDBA, or a user with DBA role granted, must execute the following code to create a new ACL:

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.create_acl (
    Acl => 'email_http_access.xml',
    Description => 'Allows access to UTL_HTTP, UTL_SMTP etc',
    Principal => 'USERNAME',
    Is_grant => TRUE,
    Privilege => 'connect',
    Start_date => SYSTIMESTAMP,
    End_date => NULL);

  Commit;
End;
/
```

The parameters, and points to note are:

- ACL - is the name of an xml file. ACLs are kept in the XML Database product, so XDB must be installed. The data is kept in the table XDB.XDB$ACL and also in a folder on the database server.
- Description - is a meaningful, please, description of what the ACL is created to allow.
- Principal - the main user account in the database which requires access to the network utilities. This may be a role or a user. The parameter is case sensitive. Beware.
- Is_grant - true grants the privilege, false denies the privilege.
- Privilege - use 'connect' for UTL_TCP, UTL_SMTP, UT:_MAIL and UTL_HTTP. Use 'resolve' for UTL_INADDR. Beware, this parameter is also case sensitive.
- Start_date and End_date are null by default. Set these to a particular TIMESTAMP to prevent the ACL from being active until or after the specific date given.
- The commit is mandatory.

## Adding Users to ACLs

Additional users and or roles are added to the ACL using the add_privilege procedure. The parameters are as above with the omission of the description parameter and the addition of the position parameter.

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.add_privilege (
    Acl => 'email_http_access.xml',
    Principal => 'OTHER_USERNAME',
    Is_grant => TRUE,
    Privilege => 'connect',
    Position => NULL,
    Start_date => NULL,
    End_date => NULL);

  Commit;
End;
/
```

The parameters and points to note are:

- Position - defines the position in the list of ACL privileges, for this principal. If an ACL higher up the list denies access and one lower down grants it again, the latter takes precedence.
- The commit is mandatory.

## Assigning ACLs to a Network Resource

Once created, the ACL must be assigned to a network using the assign_acl procedure. This is where the principal(s) which have been granted privileges in an ACL, are given access to a network resource for the duration of the ACL.

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.assign_acl (
    Acl => 'email_http_access.xml',
    Host => 'IP_or_hostname',
    Lower_port => 80,
    Upper_port => 80);

  DBMS_NETWORK_ACL_ADMIN.assign_acl (
    Acl => 'email_http_access.xml',
    Host => 'IP_or_hostname',
    Lower_port => 443,
    Upper_port => 443);

  Commit;
End;
/
```

The parameters and points to note are:

- ACL - is the name of an existing ACL xml file.
- Host - defines the network resource allowed. Only those network resources mentioned here will be allowed to be accessed. Wild cards can be used, but are advised against. See below.
- Lower_port and upper_port define the port range permitted. For default HTTP servers this will be ports 80 and 443 (for HTTPS access). For email servers, the default SMTP port is 25.
- _Only_ the specific host(s) and port(s) are accessible via the ACL. If you allow access to a web server on port 80 only, then attempting to use https on port 443 will be rejected.
- The commit is mandatory.

### Network Resources

A network can be specified as follows:

- A hostname - 'oracle.com' or 'capgemini.com'. In this case, only that hostname will be accessible from the database. If the IP address changes, it will continue working once DNS has propagated.
- An IP address - '84.37.86.172'. Again, only this network address will be accessible.
- An IP range - '84.37.86.*'. All devices on the 84.37.86 network are accessible. The '*' indicates "everything" and can be used in one or more of the 'dotted quads'.
- Everything - '*' - _best avoided_ on the grounds that someone could use your database to set up and execute attacks on any device anywhere in the world. Use the principal of _least privilege_ when setting up these ACLs.

### Ports

- By omitting the upper and lower ports, everything on the host is accessible.
- If you specify a range of port numbers then all ports in that range, inclusive, are accessible.
- If you need to set up a discontinuous port range, you must call the assign_acl procedure once for each range, as per the example above - ports 80 and 443 only are accessible.

## A Worked Example

The following example shows the user _norman_ attempting to access an HTTP web site directly to pull down a script, all from within SQL*Plus.

### Initial Test

The URL shown below is fictitious. The web site does not actually exist. The process shown below, however, _does work_ and I'm grateful to Tanel Poder for the details, which you can find [here](http://blog.tanelpoder.com/2013/01/24/sqlplus-is-my-second-home-part-7-downloading-files-via-sqlplus/?utm_source=rss&utm_medium=rss&utm_campaign=sqlplus-is-my-second-home-part-7-downloading-files-via-sqlplus "Tanel's blog post").

```sql
SQL> connect norman/secret
Connected.

SQL> set lines 1000
SQL> set trimspool on
SQL> set trimout on
SQL> set pagesize 0
SQL> set long 99999999 longchunksize 99999999
SQL> set feedback off
SQL> set head off
SQL>
SQL> spool useful_script.sql
SQL>
SQL> select httpuritype('http://fictitious.oracle.com/useful_script.sql').getCLOB()
  2> from dual;

ERROR:
ORA-29273: HTTP request failed.
ORA-06512: at "SYS.UTL_HTTP", line 1819
ORA-24247: network access denied by access control list (ACL)
ORA-06512: at "SYS.HTTPURITYPE", line 34
```

As you can see, it failed due to ACL reasons.

### Setup

The following script is executed by the SYSDBA:

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.create_acl (
    Acl => 'email_http_access.xml',
    Description => 'Allows access to UTL_HTTP, UTL_SMTP etc',
    Principal => 'NORMAN',
    Is_grant => TRUE,
    Privilege => 'connect',
    Start_date => SYSTIMESTAMP,
    End_date => NULL);

  Commit;
End;
/

BEGIN
  DBMS_NETWORK_ACL_ADMIN.assign_acl (
    Acl => 'email_http_access.xml',
    Host => 'fictitious.oracle.com',
    Lower_port => 80,	-- Default web server port
    Upper_port => 80);	-- Ditto

  Commit;
End;
/
```

### Successful Test

```sql
SQL> connect norman/secret
Connected.

SQL> set lines 1000
SQL> -- etc etc. See above.

SQL> spool useful_script.sql
SQL>
SQL> select httpuritype('http://fictitious.oracle.com/useful_script.sql').getCLOB()
  2> from dual;

-- Lots of output scrolls up the screen here ...

SQL> spool off
```

## Further Details

Further details are available from Tim Hall's web site where he takes the Oracle docs and makes sense of them. Most of the inforation given above is blatently borrowed from [Tim's excellent explanation](http://www.oracle-base.com/articles/11g/fine-grained-access-to-network-services-11gr1.php "Tim Hall's blog post on the subject. Much better than mine!").

I had to use the above today to allow a database procedure access to an email server. The application is being ported from 10g to 11g and for some unknown reason, emails started failing! Tim's blog post was extremely helpful in explaining and helping resolve the problem.

This blog post is simply a reminder to myself as to what had to be done to fix it. I'll need it again soon I expect! For the whole story, get over to Tim's web site. Don't stick around here, trust me, it's not worth it!

Thanks Tim.

Update 1st March 2013: There's an interesting [blog post here](https://dbakerber.wordpress.com/2011/06/29/11gr2-network-acl-what-a-nice-feature/ "https://dbakerber.wordpress.com/2011/06/29/11gr2-network-acl-what-a-nice-feature/") where a database trigger is used to create the ACLs after a particular user is granted execute on UTL_MAIL. You might find it interesting.
