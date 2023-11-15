---
title: "Oracle Proxy Users - What Are They Used For?"
date: "2013-08-08"
categories: 
  - "oracle"
  - "rants-raves"
---

This post has also been categorised under "rants and raves" as you will see below! Oracle 10g was the first time that proxy users could be used easily from SQL. Prior to that only Java and/or OCI programs could use them. They've been around since 8i, but not (well) documented. Want to know more? Read on....

### A Bit of Background

Many years ago, a software company I worked in - as a DBA - was taken over and we inherited a system (no names - you will see why later) which allowed numerous users the ability to use the system, and some of them got to create documents from within the application. I have no idea to this day, which version of Oracle was the first to be used for the system, but it was apparent (from discussions with _their_ technical people) that it was once a COBOL program using Indexed files as the "database". Apparently a straight conversion to Oracle was carried out, replacing each indexed file with an Oracle table.

The system was a bit of a nightmare. There were a number - at least three - of application owners in the database. Each of these had privileges and synonyms pointing to the other two, and in a few cases, User_a has a synonym that pointed to one of User_b's objects, and that turned out to be a synonym back to User_a! Go figure. As you can imaging, this did make running a full database import (only exp and imp in those days) a bit of a nightmare with all those circular references back and forth between the application owners.

The worst part, and if you are security conscious in any way, I suggest you sit down now and take a deep breath before reading on, was this. The users in the system, who were able to create documents, had the following two privileges assigned in addition to their others:

- `CREATE_ANY_PROCEDURE`
- `EXECUTE_ANY_PROCEDURE`

Yes that's correct, in order to create a document, the end users had to be able to create a procedure in the application user's schema, then execute it! End users had their own login of course, to the database, this allowed auditing to work correctly.

When I demonstrated this problem to the head of IT one day, He saw me show how it was simple for an end user to connect to the database and wipe out anything s/he desired, with only those two privileges, plus `CREATE SESSION` of course. His advice? _Do not tell the customers about this!_. I didn't.

I never did get the chance to dig down to discover the reason why the documentation enabled users had to have those abilities, unfortunately, I might have been able to suggest something else instead.

### Moving On - Proxy Users

Oracle's proxy users could have been a solution to this massive security problem. The application logged in as each end user and used the two privileges above to create a procedure in the application owner schema, executed it, then dropped the procedure again. That is how the documents were produced.

However, had the application been a little more up to date, and using Oracle 10g, we could have still had the same abilities as above, but without the need to have those nasty "ANY" privileges. Here's how we could have done it with Proxy Users.

Assume the following:

- **App_owner** is the application owner, at least, the one responsible for document creation. All document creation will be done, within this user, using procedures owned by this user.
- All other users who require to connect to the database will do so, and will be able to effectively _become_ the app_owner user, but using it as a proxy rather than logging in directly as app_owner. For the purpose of this demonstration, we shall refer to these users, collectively, as **doc_user** although there can be more than one, obvioulsy.
- We do not want to have those "ANY" privileges granted to anyone.

### Creating APP_OWNER

The application owner would be created as follows:

```sql
SQL> create user app_owner
  2  identified by app_password 
  3  default tablespace users
  4  quota unlimited on users;
User created.

SQL> create role app_owner_role;
Role created.

SQL> grant
  2    create session,
  3    create table,
  4    create procedure,
  5    create trigger
  6  to app_owner_role;
Grant succeeded.

SQL> grant app_owner_role to app_owner;
Grant succeeded.
```

In the real world, there would be more privileges granted, but these will do for now.

At this point in time, the application would be initialised by the creation of tables, procedures, functions, packages and so on. All done under the _app_owner_ user. Once the application has been set up, we can consider creating the doc_user account(s). Before we do so, we need to create a role that defines _only_ the privileges that the doc_user requires when connected to the application _as the app_owner_:

```sql
SQL> connect / as sysdba
Connected.

SQL> create role doc_user_role;
Role created.

SQL> grant 
  2    create procedure,
  3    create session
  4  to doc_user_role;
Grant succeeded.

SQL> grant doc_user_role to app_owner;
Grant succeeded.
```

Remember, we wish to connect as the _doc_user_ but _become_ the _app_owner_ for the duration of our document production. Therefore, the role we create needs to be granted to the app_owner user and not to the doc_user user(s).

In addition, because the app_owner user already has create session via the app_owner_role, you may be wondering why we also grant it to the doc_user_role. Are you wondering? I'll tell you soon. Read on!

Obviously, the role could be enhanced with other privileges, as required, to allow the application requirements to be achieved. For this demonstration, `create procedure` is enough as we need the doc_user to be able to create and execute a procedure within the app_owner schema.

### Creating DOC_USER

The application users, able to create documents, would be created as follows:

```sql
SQL> create user doc_user
  2  identified by doc_password;
User created.
```

That is all that is required. The doc_user(s) will not be creating tables etc, merely logging into the system, in this case, by becoming the app_owner and using _only_ privileges granted to that user via the doc_user_role. If the doc_users required to connect as themselves for certain parts of the application that didn't involve document production, they would obviously require the appropriate privileges, such as `create session`

As above, the real application would require some other privileges, but these will do for this demonstration.

So far, so normal. But, in order to allow the doc_user the ability to login and become the app_owner user, we need to tell Oracle that the app_owner can be connected to, through the doc_user and to only allow the privileges granted to the role doc_user_role:

```sql
SQL> alter user app_owner
  2    grant connect through doc_user
  3    with role doc_user_role;
User altered.
```

This is why we had to grant `create session` to the doc_users_role earlier. If we had not done so, we would have seen the following error when we tried to do a proxy connection:

```text
ERROR:
ORA-01045: user APP_OWNER lacks CREATE SESSION privilege; logon denied
```

If you see this, make sure that your user - app_owner in this case - has `create session` granted directly or to the role that will be enabled when proxy connections take place.

It is permitted to allow the app_owner to connect through numerous doc_users, it need not be just the one. If you have doc_user_1 through doc_user_n, then execute an `alter user` as above for each, and any of those will be able to become the app_owner for the purpose of creating documents.

### What Magic is This?

The `alter user` statement above had done two things, doc_user is now able to login using app_owner as a proxy and, when it does so, it will actually have logged in as the app_owner and will _only_ have the privileges granted to the doc_user_role available. Had we omitted the `with role` clause, doc_user would have had all the privileges of app_owner - and this is not as secure as we would like. Oracle applications and thus, users, should operate on the least _privilege_ basis.

The doc_user account doesn't even need create session any more, unless it requires to login for other reasons.

Proxy logging in is as follows:

```sql
SQL> connect doc_user[app_owner]/doc_password
Connected.

SQL> show user
USER is "APP_OWNER"
```

You will hopefully notice two things above:

- Even though the doc_user has no create session privileges, it logged in quite happily with that username and password.
- Although we logged in as doc_user, we are connected as app_owner.

You can see how the doc_user logs in, effectively as itself, but specifies the user that it wants to become after login in square brackets. Because the app_user has been told to use the role doc_user_role, then after becoming app_owner, only that role will be enabled:

```sql
SQL> select role from session_roles;

ROLE
------------------------------
DOC_USER_ROLE
```

Now, can we create a procedure? Remember, the doc_user has not been given any privileges that allow it to do so, however, the enabled role of doc_user_role does have this ability:

```sql
SQL> -- Create a "document" via a procedure.
SQL> create procedure doc_user_document
  2  as
  3  begin
  4    null;  -- This would normally do stuff to create a document.
  5  end;
  6  /
Procedure created.

SQL> -- Do the document "creation" by executing said procedure.
SQL> exec doc_user_document;

PL/SQL procedure successfully completed.

SQL> -- Tidy up again.
SQL> drop procedure doc_user_document;
Procedure dropped.
```

We can be sure that we don't have any of app_owner's other privileges active, by trying to create a table, for example:

```sql
SQL> create table test(a number);
create table test(a number)
*
ERROR at line 1:
ORA-01031: insufficient privileges
```

That looks fine – even though app_owner has create table etc, via the app_owner_role, that role isn't active when doc_user proxies in as app_owner.

### Finding Proxy Users

You can find details of proxy users in the PROXY_USERS view:

```sql
SQL> conn / as sysdba
Connected.

SQL> select * from proxy_users;

PROXY            CLIENT		 AUT FLAGS
---------------- --------------- --- -----------------------
DOC_USER         APP_OWNER       NO  PROXY MAY ACTIVATE ROLE
```

This shows you that the doc_user is a proxy user which is permitted to become the client user, app_owner, and that a role may/will be activated at login. It doesn't tell you anything about which role will be activated at login though. To discover that information, you should use either USER_PROXIES or DBA_PROXIES where the ROLE column has the details you need:

```sql
SQL> desc dba_proxies
 Name					   Null?    Type
 ----------------------------------------- -------- ----------------------------
 PROXY						    VARCHAR2(30)
 CLIENT 				   NOT NULL VARCHAR2(30)
 AUTHENTICATION 				    VARCHAR2(3)
 AUTHORIZATION_CONSTRAINT			    VARCHAR2(35)
 ROLE						    VARCHAR2(30)
 PROXY_AUTHORITY				    VARCHAR2(9)

SQL> desc user_proxies
 Name					   Null?    Type
 ----------------------------------------- -------- ----------------------------
 CLIENT 				   NOT NULL VARCHAR2(30)
 AUTHENTICATION 				    VARCHAR2(3)
 AUTHORIZATION_CONSTRAINT			    VARCHAR2(35)
 ROLE						    VARCHAR2(30)
```

As usual, DBA_PROXIES gives you details of all the proxy users in the database while USER_PROXIES only shows the ones that your currently logged in user can become, as per this example:

```sql
SQL> conn doc_user/doc_password
Connected.

SQL> select client, role from user_proxies;

CLIENT			       ROLE
------------------------------ ------------------------------
APP_OWNER		       DOC_USER_ROLE
```

We can see that our current user - doc_user - can proxy connect as the app_user with the role doc_user_role enabled. It cannot proxy login to any other database user account.

### What About Other Roles, Privileges and PL/SQL?

We have seen above that when a user is granted "connect through ... with role" then only the privileges granted to that specific role are enabled at proxy login. What about privileges granted _directly_ to the user we are becoming?

```sql
SQL> conn / as sysdba
Connected.

SQL> grant create table to app_owner;
Grant succeeded.

SQL> conn doc_user[app_owner]/doc_password
Connected.

SQL> create table test(a number);
Table created.
```

Whoops! So, privileges granted _directly_ to the app_owner are _also_ enabled when we proxy login to it, even though a role was specified. This could be something to consider when setting up your proxy users.

Who owns the new table then?

```sql
SQL> select owner from all_tables where table_name = 'TEST';

OWNER
------------------------------
APP_OWNER
```

So, that's another thing to consider, when you have become another user via a proxy login, the other user owns any objects you create. As owner, privileges to `INSERT`, `DELETE`, `UPDATE` and `SELECT` will also exists on these tables, as well as any other that it owns and to which `SELECT` etc may not have been granted to the doc_user, in this case.

Bear in mind also, that roles enabled in a session are _disabled_ when PL/SQL is being compiled or executed.

Have fun!

### Terminology

In the discussions above, I've tended to stay away from the various terminology that Oracle uses in an effort to try and make things a little more clear. However, before I go, here's the information you may require:

- Proxy User - is the user that is allowed to become another user. In the above, the proxy user is the doc_user as it is permitted to become the app_owner.
- Client User - is the user that the proxy user is allowed to become. In the above, the client user is app_owner.
- Proxy Login - is a special format connect string where the proxy user's name and password is used, as normal, but with the addition of the client user's name in square brackets after the proxy user's name.
    
    connect proxy_user[client_user]/proxy_user_password@.....
