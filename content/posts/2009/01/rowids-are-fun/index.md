---
title: "ROWIDs are fun"
date: "2009-01-26"
categories: 
  - "oracle"
---

In a previous posting here on the subject of Lazy Developer Syndrome, I showed a small fragment of code where I `SELECT`ed the ROWID in addition to all the other data I wanted, then `UPDATE`d the same row using the ROWID I had stored rather then using the Primary Key index that I used to `SELECT` the row in the first place. Why did I do this?

Selecting the ROWID is always a good idea if you intend to `UPDATE` or indeed, `DELETE`, it afterwards.  The ROWID is after all the fastest way to get to a row in your table. It's how the indexes get you to the data after all.

If you have a primary key constraint and the index supporting it has a height of 2, then to get from your supplied primary key value to the data requires 3 single block accesses - the index root and the correct leaf block. From the leaf block we get the ROWID for the requested primary key value, and a further single block read takes place to fetch back the block containing the required row.

Some of these blocks may be in the cache but then again, they might all  not be - so 3 physical reads would be required as the worst case and 3 logical reads as a best case.

An `UPDATE` specifying the primary key value again will also have to perform 3 more single block reads, hopefully from the cache. However, an `UPDATE` using the ROWID already fetched will only have to perform one block read, again hopefully from the cache, and that block will be the actual data block to be amended. We don't need to use the index and **we save 2 single block reads**. Good eh?

A `DELETE` using a ROWID or a primary key value will also take differing numbers of block reads. If you delete using the ROWID, the index has to be read and updated by removing the entry for the row in question and if you `DELETE` using the Primary Key value, the index will be read again (in consistent mode) and the data block will be read in current mode in order to make the DELETE work.

My own testing, so far, has shown that the difference is a single block read saved when using the ROWID rather than the Primary key value however, that figure will change if the height of the index itself changes.

#### Update by Primary Key.

```sql
SQL> update indextest 
     set name = name 
     where id = 86057;
1 row updated.
```
```text
Execution Plan
----------------------------------------------------------
 0 UPDATE STATEMENT Optimizer=ALL_ROWS 
    (Cost=2 Card=1 Bytes=79)
 1 0 UPDATE OF 'INDEXTEST'
 2 1  INDEX (UNIQUE SCAN) OF 'INDEXTEST_PK' (INDEX (UNIQUE)) 
       (Cost=1 Card=1 Bytes=79)

Statistics
-----------------
1 db block gets
2 consistent gets
0 physical reads
1 rows processed
```

#### Update by ROWID.

```sql
SQL> update indextest 
     set name=name 
     where rowid = 'AAAXktAAEAAAET8ABI';
1 row updated.
```
```text
Execution Plan
----------------------------------------------------------
 0 UPDATE STATEMENT Optimizer=ALL_ROWS 
   (Cost=1 Card=1 Bytes=78)
 1 0 UPDATE OF 'INDEXTEST'
 2 1 TABLE ACCESS (BY USER ROWID) OF 'INDEXTEST' (TABLE) 
       (Cost=1 Card=1 Bytes=78)

Statistics
-----------------
1 db block gets
0 consistent gets
0 physical reads
1 rows processed
```

#### Delete by Primary Key.

```sql
SQL> delete from indextest 
     where id = 86057;
1 row deleted.
```
```text
Execution Plan
----------------------------------------------------------
 0 DELETE STATEMENT Optimizer=ALL_ROWS 
    (Cost=1 Card=1 Bytes=13)
 1 0  DELETE OF 'INDEXTEST'
 2 1   INDEX (UNIQUE SCAN) OF 'INDEXTEST_PK' (INDEX (UNIQUE)) 
        (Cost=1 Card=1 Bytes=13)

Statistics
-----------------
5 db block gets
2 consistent gets
0 physical reads
1 rows processed
```
```sql
SQL> rollback;
Rollback complete.
```

#### Delete by ROWID.

```sql
SQL> delete from indextest 
     where rowid = 'AAAXktAAEAAAET8ABI';
1 row deleted.
```
```text
Execution Plan
----------------------------------------------------------
 0 DELETE STATEMENT Optimizer=ALL_ROWS 
   (Cost=1 Card=1 Bytes=25)
 1 0  DELETE OF 'INDEXTEST'
 2 1   TABLE ACCESS (BY USER ROWID) OF 'INDEXTEST' (TABLE) 
        (Cost=1 Card=1 Bytes=25)

Statistics
-----------------
5 db block gets
1 consistent gets
0 physical reads
1 rows processed
```
```sql
SQL> rollback;
Rollback complete.

SQL> exit
```

So, when creating your application, the idea is to build in performance from the start (ie, you do not add it in after the fact!) so when reading data that will be updated or deleted in a subsequent operation, always grab hold of the ROWID and use that to make your DELETE or UPDATE as efficient as you can.

By the way, never (can we say 'never' in these politically correct times?) store the location of a row in another table as a ROWID - if you do, and you ever export the table and reimport it, the ROWIDs you carefully saved will no longer point to the row that you thought they did!

Did you never wonder why indexes are stored in the export file as a CREATE INDEX statement, and not as the pure index data itself?

Cheers.
