---
title: "PL/SQL expression evaluator"
date: "2009-07-02"
categories: 
  - "oracle"
---

The following is a pretty nice expression evaluator for Oracle's PL/SQL language. You pass it a string containing an expression that would return a numeric value when evaluated and it will evaluate the entire expression and return the number.

The passed expression must be in valid Oracle syntax or you will get a NULL instead of a number.

```sql
CREATE FUNCTION expression (iExpression IN varchar2)
RETURN number AS

vResult number;

BEGIN
-----------------------------------------------------------
-- Expression evaluator for PL/SQL.
-- Pass in a string containing the expression you want to
-- evaluate using correct Oracle syntax and the result will
-- be returned.
--
-- WARNING: Causes a parse for every expression and will
-- soon fill your cache with similar statements.
-----------------------------------------------------------
-- Norman Dunbar    02 July 2009        Created new function.
-----------------------------------------------------------

execute immediate 'begin :r := ' || iExpression ||
                          ' ; end;' using OUT vResult;

RETURN vResult;

EXCEPTION
WHEN others THEN
RETURN NULL;  -- It all went horribly wrong!
END;
/
```

Of course, it has its drawbacks, the biggest one being that it will, if called repeatedly with different expressions, fill up your cache with similar statements.

So my challenge is to come up with a similar and equally short PL/SQL routine that will not age potentially useful SQL out of the cache while leaving multiple copies of similar SQL statements such as:

```sql
begin :r := 2+2; end;
```

and so on lying around taking up valuable cache space.

Cheers.
