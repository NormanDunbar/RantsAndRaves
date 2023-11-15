---
title: "Interesting Foible with Oracle Dates"
date: "2016-09-08"
categories: 
  - "oracle"
---

I have a table with dates in, and some NULLs. Two people, on the same database, running the same `SELECT` query, in the same schema, with the same privileges, get vastly differing results. Why? Fine Grained Auditing is not at play here.

_Table names, column names etc have been changed to protect the guilty._

In the table in question, the `A_DATE` column is correctly defined as `DATE`, rather than anything else unsuitable.

I have boiled the problem down to the following code. The table I'm using has 76 rows of which, 27 are NULLs in the `A_DATE` column. It took a while to notice the bug in the code though, maybe I should do some more development work?

```sql
select count(\*) from norm 
where 
trim(nvl(a_date, to_date('07/04/1960','dd/mm/yyyy'))) = to_date('07/04/1960','dd/mm/yyyy');
```

It looks ok, it does not give any errors, but running it gives inconsistent results depending on the setting of `NLS_DATE_FORMAT`:

```sql
alter session set nls_date_format='dd/mm/yyyy';

select count(\*) from norm 
where 
trim(nvl(a_date, to_date('07/04/1960','dd/mm/yyyy'))) = to_date('07/04/1960','dd/mm/yyyy');

COUNT(\*)
-------
     27
```

```sql
alter session set nls_date_format='dd-mon-rr';

select count(\*) from norm 
where 
trim(nvl(a_date, to_date('07/04/1960','dd/mm/yyyy'))) = to_date('07/04/1960','dd/mm/yyyy');

COUNT(\*)
-------
     27
```

```sql
alter session set nls_date_format='dd-mon-yy';

select count(\*) from norm 
where 
trim(nvl(a_date, to_date('07/04/1960','dd/mm/yyyy'))) = to_date('07/04/1960','dd/mm/yyyy');

COUNT(\*)
-------
      0
```

WTH? Zero? Really?

```sql
alter session set nls_date_format='dd-mon-yyyy';

select count(\*) from norm 
where 
trim(nvl(a_date, to_date('07/04/1960','dd/mm/yyyy'))) = to_date('07/04/1960','dd/mm/yyyy');

COUNT(\*)
-------
     27
```

```sql
alter session set nls_date_format='mm-dd-yyyy';

select count(\*) from norm 
where 
trim(nvl(a_date, to_date('07/04/1960','dd/mm/yyyy'))) = to_date('07/04/1960','dd/mm/yyyy');

COUNT(\*)
-------
     27
```

So, what's going on here? Well, it seems from the docs that the `TRIM()` function is not really supposed to be applied to dates, but Oracle doesn't complain. It returns a `VARCHAR2` value, and not a `DATE` value as the code _appears_ to return.

This `VARCHAR2` is then compared with a `DATE` value given on the right side of the '=', so there is a bit of implicit conversion going on, and _I'm positive_ that the `DATE` is converted to a `VARCHAR2` for the comparison, and this is a bad way to compare `DATE` values, as `VARCHAR2`s. After all, 07/04/1960 is bigger than 01/05/2016 isn't it? (No, it isn't, well, not as a `DATE`, but as a `VARCHAR2` ...)

_Some_ of the other non-null dates in the table are:

```
17/03/2016
11/12/2015
02/12/2014
30/10/2014
29/10/2014
02/10/2013
14/10/2009
08/07/2008
03/07/2008
24/06/2008
05/06/2008
```

The fix? Obvious really, the developer intended to use `TRUNC()` but mysteriously typed `TRIM()` instead. Once changed, it "just" worked - for all known values of `NLS_DATE_FORMAT`!
