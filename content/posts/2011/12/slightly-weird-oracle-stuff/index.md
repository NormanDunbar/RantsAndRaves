---
title: "Slightly Weird Oracle Stuff"
date: "2011-12-14"
categories: 
  - "oracle"
---

I knew you could do this:

```sql
SQL> select 1234567890 as abc from dual;

       ABC
----------
1234567890
```

or

```sql
SQL> select 1234567890 abc from dual;

       ABC
----------
1234567890
```

But I didn't know that this worked as well -- there's no space between the value and the alias name:

```sql
SQL> select 1234567890abc from dual;

       ABC
----------
1234567890
```

So I did a bit of playing and discovered that there is a difference if the alias is D or F but no other single character:

```sql
SQL> select 1234567890d, 1234567890f, 1234567890p from dual;

1234567890D 1234567890F          P
----------- ----------- ----------
 1.235E+009  1.235E+009 1234567890
```

This shows the values in Scientific notation when D or F is used as an alias in this manner, but not if used in this manner:

```sql
SQL> select 1234567890 d, 1234567890 f, 1234567890 p from dual

         D          F          P
---------- ---------- ----------
1234567890 1234567890 1234567890
```

Then it gets stranger, note the alias names and the corresponding column names:

```sql
SQL> select 1234567890df from dual;

         F
----------
1.235E+009

SQL> select 1234567890fd from dual

         D
----------
1.235E+009

SQL> select 1234567890fa from dual

         A
----------
1.235E+009
```

I get the impression that a trailing F or D on a number means "display as floating point or decimal" then the F/D is dropped and the A used as a label. I can't find this in the docs though.

Works with strings as well but the F/D thing doesn't appear with strings. Doesn't work - for obvious reasons - with column names.

**UPDATE**: Thanks to Maxim on the Oracle-L list, the answer is [~~here~~](http://docs.oracle.com/cd/E11882_01/server.112/e26088/sql_elements003.htm#sthref357 "http://docs.oracle.com/cd/E11882_01/server.112/e26088/sql_elements003.htm#sthref357") (Sorry, another dead link!)

**UPDATE 2**: Thanks to Jonathan Lewis also on the Oracle-L list, it seems that Tanel Poder has also come across this. On his blog [here](http://blog.tanelpoder.com/2011/01/10/is-this-valid-sql-syntax/ "http://blog.tanelpoder.com/2011/01/10/is-this-valid-sql-syntax/").

Cheers,
