---
title: "Swap 2 Values, in SQL, Without Using a Temporary Variable"
date: "2013-04-24"
categories: 
  - "oracle"
---

Recently, I saw a mention of an interview question for SQL developers. It was something along the lines of:

{{< alert theme="success" >}}
There is a table with a sex column. It has been discovered that the values are swapped around and need to be corrected. How would you swap all 'M' values to 'F' and all 'F' values to 'M', while leaving the other values untouched, __in one single SQL statement and without requiring the use of any temporary variables__.
{{< /alert >}}

So, how would you do it? Here's my example.

First, the before look:

```sql
select sex, forename from bedrock;

S  FORENAME
- --------
M  Wilma
   Dino
M  Betty
F  Fred
F  Barney
U  Pebbles

6 rows selected.
```

I think we can safely say that the sex column is somewhat back to front. My solution was to use the `CASE` statement in an SQL `UPDATE` command:

```sql
update bedrock
set sex =
  case sex 
    when 'F' then 'M'
    when 'M' then 'F'
  end;
```

Finally, the after look:

```sql
select sex, forename from bedrock;

S  FORENAME
- --------
F  Wilma
   Dino
F  Betty
M  Fred
M  Barney
U  Pebbles

6 rows selected.
```

It appears to be safe to commit!

Using the `CASE` statement is useful. In the old days, we would need something like the following:

```sql
Update bedrock set sex = 'T' where sex = 'M';
Update bedrock set sex = 'M' where sex = 'F';
Update bedrock set sex = 'F' where sex = 'T';
```

For huge tables, that could have taken a while, and it's highly unlikely that a sex column would be indexed, so three full table scans would have been the order of the day.

Using `CASE` we get away with a single scan, plus, the statement _short circuits_ when it hits the first matching `WHEN` clauses. So, if the first sex value was 'F' it would be changed to 'M' however it would then stop checking and would _not_ change the newly set 'M' back to an 'F'.
