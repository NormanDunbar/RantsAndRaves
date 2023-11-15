---
title: "Spatial Indexes and Oracle Errors. How to fix."
date: "2013-05-20"
categories: 
  - "oracle"
---

If, like me, you have suffered from _ORA-29902 Error in executing ODCIIndexStart() routine errors_ where Spatial indexes are involved, the following might help you fix it.

The error involved in the following has been extracted from a log file for a system which doesn't use Spatial or Locator itself, but calls out to a separate database which does have Locator installed. This latter database was created using Transportable Tablespaces, exported from 10.2.0.5 Enterprise Edition on HP-UX and imported into 11.2.0.3 Standard Edition on Linux x86-64.

There were a number of errors creating a few of the spatial indexes on tables, like the one that follows in the example, that had zero rows in them. Oracle Support assured us that this was not a problem. And we believed them. Sigh!

## The Problem

The following query demonstrates the problem.

```sql
CONNECT CADDBA/password

SELECT * FROM TEXT_FORESHORE A
WHERE MDSYS.SDO_RELATE( A.GEOM, 
                        MDSYS.SDO_GEOMETRY(2003,81989,
                        NULL,
                        MDSYS.SDO_ELEM_INFO_ARRAY(1,1003,3),
                        MDSYS.SDO_ORDINATE_ARRAY(362000,600000,363000,601000)),
                        'MASK=ANYINTERACT QUERYTYPE=WINDOW') = 'TRUE';

*
ERROR at line 1:
ORA-29902: error in executing ODCIIndexStart() routine
ORA-13203: failed to read USER_SDO_GEOM_METADATA view
ORA-13203: failed to read USER_SDO_GEOM_METADATA view
ORA-06512: at "MDSYS.SDO_INDEX_METHOD_10I", line 333
```

## Working Out

I am definitely not a Spatial guru, but the above doesn't look right to me. Looking at Google, the problem is caused by the Spatial Index being not there, missing, absent. Ok, let's create it.

```sql
CREATE INDEX IDX_T142_GEOM ON TEXT_FORESHORE(GEOM)
INDEXTYPE IS MDSYS.SPATIAL_INDEX
PARAMETERS('TABLESPACE=CAD_PRSN_IDX_SPAT SDO_RTR_PCTFREE=0')
NOPARALLEL;

CREATE INDEX IDX_T142_GEOM ON TEXT_FORESHORE
                    *
ERROR at line 1:
ORA-00955: name is already used by an existing object
```

Ok, to me, that says that the index is actually present. `DBA_INDEXES` shows this to be the case. Apparently, it needs to be dropped and recreated, so I carry on:

```sql
DROP INDEX IDX_T142_GEOM ;
Index dropped.

CREATE INDEX IDX_T142_GEOM ON TEXT_FORESHORE(GEOM)
INDEXTYPE IS MDSYS.SPATIAL_INDEX
PARAMETERS('TABLESPACE=CAD_PRSN_IDX_SPAT SDO_RTR_PCTFREE=0')
NOPARALLEL;

CREATE INDEX IDX_T142_GEOM ON TEXT_FORESHORE
*
ERROR at line 1:
ORA-29855: error occurred in the execution of ODCIINDEXCREATE routine
ORA-13203: failed to read USER_SDO_GEOM_METADATA view
ORA-13203: failed to read USER_SDO_GEOM_METADATA view
ORA-06512: at "MDSYS.SDO_INDEX_METHOD_10I", line 10
```

Aha. Something different this time. Still not working though. It might be as simple as the CADDBA user not having the correct privileges. Create table and create sequence is required to create a spatial index - whether directly in the schema or as another user creating on in the schema in question. So:

```sql
CONNECT / AS SYSDBA

SELECT PRIVILEGE
FROM DBA_SYS_PRIVS
WHERE PRIVILEGE IN ('CREATE TABLE', 'CREATE SEQUENCE' )
AND GRANTEE = 'CADDBA';

PRIVILEGE
---------------
CREATE SEQUENCE
CREATE TABLE

2 rows selected.
```

So that's not the problem this time. Looking into the `USER_SDO_GEOM_METADATA` view, for this user (every user with Spatial data should have this view) I see nothing for this table_name and column_name:

```sql
CONNECT CADDBA/password

SELECT * FROM USER_SDO_GEOM_METADATA
WHERE TABLE_NAME = 'TEXT_FORESHORE'
AND COLUMN_NAME = 'GEOM';

no rows selected
```

Ok, a clue. I (vaguely) know that in order to create a spatial index, that view needs some data telling it all about the column in question. As this database had been created from a legacy database (which very very rarely gets updated) I was ok to extract the data from legacy and insert it directly here.

Did I mention, each time the commands fail to create the index in question, they create the index in question? So after each failure, you have to drop it again. Sigh!

```sql
DROP INDEX CADDBA.IDX_T142_GEOM ;
Index dropped.

INSERT INTO USER_SDO_GEOM_METADATA
VALUES ('TEXT_FORESHORE','GEOM',
        mdsys.SDO_DIM_ARRAY(
             mdsys.SDO_DIM_ELEMENT('Easting', 0, 700000, .0005),
             mdsys.SDO_DIM_ELEMENT('Northing', 0, 1300000, .0005)
        ), 81989);

1 row created.

COMMIT;
Commit complete.
```

Now can I create the index?

```sql
CREATE INDEX IDX_T142_GEOM ON TEXT_FORESHORE(GEOM)
INDEXTYPE IS MDSYS.SPATIAL_INDEX
PARAMETERS('TABLESPACE=CAD_PRSN_IDX_SPAT SDO_RTR_PCTFREE=0')
NOPARALLEL;

Index created.
```

And success at long last. Spatial, I hate you! Does the query work now?

```sql
SELECT * FROM TEXT_FORESHORE A
WHERE MDSYS.SDO_RELATE( A.GEOM, 
                        MDSYS.SDO_GEOMETRY(2003,81989,
                        NULL,
                        MDSYS.SDO_ELEM_INFO_ARRAY(1,1003,3),
                        MDSYS.SDO_ORDINATE_ARRAY(362000,600000,363000,601000)),
                        'MASK=ANYINTERACT QUERYTYPE=WINDOW') = 'TRUE';

no rows selected
```

After all that work, no rows selected is exactly the correct answer. The table is empty, so I would have been very surprised to see anything other than that response.

## The Solution

The solution to my particular problem was to:

- Drop the so called missing index.
- Make sure correct data is in `USER_SDO_GEOM_METADATA` for the table and column in question. Each user with Spatial data will have one of these views, so you need to be in the appropriate user.
- Create the index again.
- Test the failing query, and it should work.

Cheers.
