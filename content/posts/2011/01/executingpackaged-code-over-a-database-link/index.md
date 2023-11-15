---
title: "ExecutingPackaged Code Over a Database Link"
date: "2011-01-18"
categories: 
  - "oracle"
---

Ever wondered how you can call a packaged procedure (or function) which resides at the far end of a database link? Wonder no more! 

```sql
begin 
    package_name.Procedure_name@db_link_name(parameter, parameter, ...); 
end;
```

Calling a function is just as easy: 

```sql
declare vResult number(10); 
begin 
    vResult := package_name.Function_name@db_link_name(parameter, parameter, ...); 
end;
```

Unfortunately, you don't appear to be able to do this sort of thing via a synonym:

```sql
create or replace synonym X for package_name@db_link_name; 

begin 
    X.Procedure_name(parameter, parameter, ...); 
end;
``` 

You get **ORA-00904: "SQLTEST"."TEST": invalid identifier** instead of a result. :-(

**Update: 20 January 2011**: If you prefix the remote object name with the remote object owner, _regardless_ of the fact that the database link is connecting to that schema anyway, then you _can_ subsequently call the function, procedure or packaged code using the synonym. Result - thanks to Oracle Support.

```sql
create or replace synonym X for owner.package_name@db_link_name; 

begin 
    X.Procedure_name(parameter, parameter, ...); 
end;
``` 

It "just" works. :-)

Cheers,
Norm.
