---
title: "Oracle's Deferred Segment Allocation Breaks Transportable Tablespace Imports."
date: "2012-11-14"
categories: 
  - "oracle"
---

In order to downgrade an 11.2.0.3 Enterprise Edition database to Standard Edition, I had to use a Transportable Tablespace export/import. Because the default setting for `DEFERRED_SEGMENT_CREATION` is `TRUE`, the tablespace import barfed with numerous "_IMP-00017: following statements failed with ORACLE error 1647:_" errors. Want to know why?

The ORA-01649 error is "_Tablespace is read only, cannnot allocate space in it_" which is interesting as I'm importing a Transportable Tablespace dump file and _all_ the tablespaces are read only after being created, until I manually make then read write.

In the source database, the "broken" tables all have zero rows in them, and have no entry in DBA_SEGMENTS which means that the _Deferred Segment Allocation_ feature has indeed done its stuff, and deferred allocating a segment until the first row of data is entered (but not necessarily committed!) into the table.

{{< alert theme="info" >}}
The `DEFERRED_SEGMENT_CREATION` is also defaulted to `TRUE` in Standard Edition databases, but the parameter has _no effect_ in these, as the following shows:
```sql
Connected to:
Oracle Database 11g Release 11.2.0.3.0 - 64bit Production

SQL> show parameter deferred_seg

NAME                       TYPE     VALUE
-------------------------- -------- -------------
deferred_segment_creation  boolean  TRUE

SQL> create table test(a number);
Table created.

SQL> select table_name, segment_created
  2  from user_tables
  3  where table_name = 'TEST';

TABLE_NAME           SEG
-------------------- ---
TEST                 YES
```
So, even with an empty table, in Standard Edition, the deferred segment allocation does not take place. This is exactly why I'm having problems running a transportable tablespace import from an Enterprise Edition database to a Standard Edition one. It seems that Standard expects everything to have at least one allocated segment.
 
If the importing database is Enterprise Edition, this problem doesn't occur.
{{< /alert >}}

The quick workaround is to run a table based export, using `exp` or `expdp`, of the affected tables, and import that at the receiving database using `imp` or `impdp`.

The longer term workaround is to make sure that `DEFERRED_SEGMENT_CREATION` is set to `FALSE` in the spfile.

In the meantime, I'm logging a bug with Oracle as the `DBMS_TTS.TRANSPORT_SET_CHECK` procedure should identify these tables and warn about them, or, the import should correctly import them anyway.
