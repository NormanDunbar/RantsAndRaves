---
title: "Dropping Temporary Tables  (With Bonus, Broken Check Constraints!)"
date: "2016-08-09"
categories: 
  - "oracle"
---

I found a broken check constraint, one that simply wouldn't work, on a database. It was created as:

```sql
... CHECK(COLUMN_NAME IN ('Y','N',NULL)) ;
```

Try it yourself, it doesn't work! Anyway, I needed to find if there were any other check constraints broken in this manner, so I did the following:

```sql
    select owner, 
           table_name, 
           constraint_name, 
           to_lob(search_condition) search_condition
    from   dba_constraints
    where  owner = 'XXXXX' 
    and    constraint_type = 'C'
    and    upper(search_condition) like '%IN%,%NULL%'
    order  by 1,2,3;    
```

Of course, that barfed because the `SEARCH_CONDITION` column is a `LONG` data type. Sigh! I thought those things were deprecated! Never mind, I did this next:

```sql
-- Can't filter search_condition as it's a LONG data type.
create global temporary table check_constraints on commit preserve rows
as (
    select owner, 
           table_name, 
           constraint_name, 
           search_condition
    from   dba_constraints
    where  owner = 'XXXXX' 
    and    constraint_type = 'C'
) ;

select \* from check_constraints
where upper(search_condition) like '%IN%,%NULL%';

-- Do other meaningful stuff here ....
-- Then eventually ...

drop table check_constraints;
```

But the `DROP TABLE` command resulted in an error `ORA-14452: attempt to create, alter or drop an index on temporary table already in use`. Hmm!

First of all, I had no indexes, so the message is only slightly misleading, but regardless, I couldn't drop my temporary table when I was finished with it.

The solution is amazingly simple:

```sql
truncate table check_constraints;
```

After that, dropping the table "just works".

And yes, there were quite a few broken check constraints. Duvelopers!

What's Broken? Well, if you create a check constraint as per the one listed back at the start of this eRant, you will not see any errors from Oracle. Nor will you see any errors when you `INSERT` or `UPDATE` rows with invalid values in the column. It's that `NULL` thing that kills your constraint. It simply means that _any_ value you have in the column will be accepted.

```sql
drop table test cascade constraints purge;

create table test(a varchar2(1));

alter table test add constraint chk_a
check (a in ('Y','N', NULL));

insert into test(a) values ('N');
insert into test(a) values ('Y');
insert into test(a) values (NULL);
insert into test(a) values ('T');
```

Huh? That last one couldn't have worked, could it?

```sql
select A from test;

 A
--
 N
 Y

 T
```

Yup, the constraint is indeed useless.

Have fun!
