---
title: "It must be efficient, I'm using bind variables!"
date: "2009-02-04"
categories: 
  - "oracle"
---

For many years, various big guns - and a lot of smaller ones - in the Oracle world have been advocating, nay demanding, that we \[almost\] always use bind variables in our SQL code. The reason is simple, it's shareable, efficient,  reduces parsing and allows the application to scale up to more and more users.

Over the years I have spent fixing and tuning Oracle databases, I have noticed a trend of developers moving away from hard coding everything to using binds more and more in their code. How refreshing, the performance must also be improving, mustn't it?

Not quite! :-(

The problem is becoming apparent that a lot of developers don't know anything/much about Oracle and also don't know anything/much about their own chosen language either.

I'm pleasantly surprised when I see code like the following in the cache on my databases :

```sql
SELECT stuff FROM some_table 
WHERE some_column = :a_bind_variable;
```

It's a start, but it is still broken. How can this be?

While the  developer is indeed using binds, s/he has not considered how that statement is created within the application. As there are many application coding languages, I shall resort to my own pseudo-code to demonstrate the problem.

```sql
Function GetEmpForID(int empId, Connection conn)
begin
 Statement Stmt(conn, "Select stuff from some_table 
                       where some_column = :some_value");
    Stmt.Parse();
    Stmt.Bind(":some_value", empId);
    ResultSet Rslt =  Stmt.Execute();
    //Process Results here
end
```

Looks great doesn't it? Well, no, it doesn't. The major problem is that fact that the statement will be parsed each and every time that this function is called. Within a function (in most languages) the statement is created on the stack and on exit from the function, deleted.

In reality, we don't get much in the way of efficiency improvements because we still have a 1:1 parse:execute ratio, not what we want at all. What we need to do is make the statement external to the function (global or whatever the language supports) similar to the following Java-ish *pseudo-code*.

```Java
Connection conn;
Statement StmtEmpForId(conn, "Select stuff from some_table 
                       where some_column = :some_value");

Function GetEmpForID(int empId)
begin
    external Statement StmtEmpForId;
    if not StmtEmpForId.Parsed() then StmtEmpForId.Parse();
    StmtEmpForId.Bind(":some_value", empId);
    ResultSet Rslt =  StmtEmpForId.Execute();
    //Process Results here
end
```

Now we are getting somewhere! The statement is external to the function itself, so exists even when the function has exited. If this is the first time that we call the function, the statement will be found to be unparsed, so we can parse it - this avoids the application parsing statements at startup and possibly parsing statements that never get used.

Once parsed, and on every subsequent call of the function, all we have to do is bind the variable and execute the statement before processing the results.

As long as the statement (and connection)  remain in scope (hence being globals) then we only parse the statements once regardless of the number of times that we execute it. So, if we execute our one statement a million times, we have only carried out one single \[hard\] parse.

In C++ you would, I suppose, create an appConnection object with members for the Oracle (OCCI?) Connection object and each of the Statements you wish to use in the application, after all, the statements should remain in the same scope as the connection shouldn't they?

In Java, well, I have no idea, all the Java stuff I've looked at seems to be reinventing the wheel over and over again - who knows what those guys get up to! ;-)

Cheers.
