---
title: "So, How Do You Change a User's Password"
date: "2013-11-17"
categories: 
  - "oracle"
---

The Oracle database allows the users to change their passwords as follows:

SQL> ALTER USER me IDENTIFIED BY my_new_password;

or, alternatively, to use the `PASSWORD` command, which prompts for the old and new passwords.

Of course, if the user has forgotten their old password, the system manager can do the necessary:

SQL> ALTER USER forgetful_user IDENTIFIED BY a_new_password;

Now, if there are profiles in use, as there are, and these profiles have a password verification function defined, these passwords will be validated to ensure that they adhere to the installation standards.

Sadly, all is not as it seems.

The verification function is passed three parameters:

- Username
- Old Password
- New Password

In 12c, the standard verification function has an inbuilt helper function called `string_distance` which determines how different the new password is from the old one. The problem is, regardless of what you have set for that difference to be, it is not always executed. The code in the verification function, $ORACLE_HOME/rdbms/admin/utlpwmg.sql, has something resembling this check in it:

```sql
    if old_password is not null then
        result := string_distance(old_password, new_password)
        ...
```

Interesting, if the old password is NULL? How can this be? Well, in reality, it is simple. Here are the cases where the old password _will not_ be NULL:

- When the user calls the `PASSWORD` function.
- When the user executes `ALTER USER me IDENTIFIED BY my_new_password REPLACE old_password;`

And here are the cases when the old password _will_ be NULL:

- When the user executes `ALTER USER me IDENTIFIED BY my_new_password;`
- When the SYS user executes `ALTER USER forgetful_user IDENTIFIED BY a_new_password;`
- When the SYS user executes `ALTER USER forgetful_user IDENTIFIED BY a_new_password REPLACE old_password;`

And thereby hangs the rub. If the SYSDBA always changes the passwords, then the old password is always NULL, and some of the verification checks will not be carried out. _Only_ when the user affected changes the password using the `PASSWORD` command or passes the `REPLACE` clause to the `ALTER USER` command, will the old password be supplied to the verification function.

It actually makes sense, if you think about it, the password is stored, by Oracle, as the result of a one-way hash. This means that there is no way to retrieve the plain text password from the hashed value. However, it appears that regardless of whether the SYSDBA user supplies the old password in the `ALTER USER ... REPLACE` command, it is _not_ passed through to the verification function.

Just a little something to watch out for as it can allow your users to get past some of the checks in the verification function - adding a one character suffix to the old password.for example, can get past the checks if the old password is not supplied.

What do you mean, you never knew there was a `REPLACE` clause on the `ALTER USER` command? ;-)
