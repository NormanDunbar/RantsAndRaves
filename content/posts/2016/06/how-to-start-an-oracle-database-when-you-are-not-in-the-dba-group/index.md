---
title: "How to Start an Oracle Database When You Are Not in the DBA Group"
date: "2016-06-24"
categories: 
  - "linux"
  - "oracle"
  - "windows"
---

This applies to Linux, Unix as well as Windows, but affected me on a Windows 2012 Server running Oracle 11.2.0.4 Enterprise Edition.

My user on the server was an administration user, but not in the `ora_dba` group. This is required to `connect / as sysdba` within SQL\*Plus. The SYS password had been changed recently but whoever did it, did not update the password vault. The users were urgently requiring their database be started, I was the only DBA in the office, the SYS password was unknown, and my user didn't belong _directly_ to the `ora_dba` group. What to do?

It's not quite the dreadful hack that the title of this post may indicate. Depending on the setup for the server, you may need administrator rights to move files around. Plus, most importantly, you do need to know the SYS password for at least one database on the server.

My user account was indirectly a member of the `ora_dba` group, via my administrator rights, but it seems I need to be directly a member of the group to login `/ as sysdba`.

That said, the short, bullet point method is as follows:

- `Cd %ORACLE_HOME%\database`
- `set ORACLE_SID=dbadb01`.
- Rename the current password file `pwddbadb01.ora` to `pwddbadb01.ora.keep`.
- Copy another password file, for which _I did know_ the SYS password, to `pwddbadb01.ora`.
- `Sqlplus sys/known_password as sysdba`.
- `startup`.
- `exit`.
- Delete `pwddbadb01.ora`.
- Rename `pwddbadb01.ora.keep` to `pwddbadb01.ora`.

This way I got the database started, the users were happy, and I made sure I got the password vault updated to save me this grief next time!
