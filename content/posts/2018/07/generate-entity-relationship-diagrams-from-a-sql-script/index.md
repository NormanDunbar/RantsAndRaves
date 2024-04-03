---
title: "Generate Entity Relationship Diagrams from a SQL Script."
date: "2018-07-12"
categories: 
  - "oracle"
---

_Sometimes_, just _occasionally_, you find yourself as a DBA on a site where, for some strange and unknown reason, you don't have an Entity Relationship Diagram (ERD) for the database that you are working on. You _could_ use a tool such as `Toad`, or `SQL*Plus` (or even, `SQL Developer` - if you must) to generate a list of referential integrity constraints. There has to be a better way.

The problem with lists is, they are just words. And they do say that a picture is worth a thousands words, so lets do pictures.

### Graphviz

`Graphviz` is a set of tools for visualising graphs (that's _directed_ or _undirected_ graphs as opposed to _Cartesian_ graphs by the way) and the source files for the utility are simple text files. So, can we generate an ERD in text format and have `graphviz` convert it to an image? Of course we can. But first, get thee hence to [https://graphviz.gitlab.io/download/](https://graphviz.gitlab.io/download/) and download the utility for your particular system - it runs cross platform and is free.

### Generating Source

The following query will generate a list of parent -> child lines in the output, that show the relationship between any pair of tables, given a suitable starting owner and table\_name.

```sql
\-------------------------------------------------------------
-- Generate a simple "dot" file to be processed with GraphViz
-- to create an image showing the referential integrity
-- around a single table.
--
-- Norman Dunbar
-------------------------------------------------------------
set lines 2000 trimspool on trimout on
set pages 2000
set echo off
set feed off
set verify off
set timing off
set head off

accept owner_name prompt "Enter owner: "
accept table_name prompt "Enter table name: "

spool refint.dot

select '// dot -Tpdf -o this_file.pdf this_file.dot' || chr(10) || chr(10) ||
'digraph refInt {' || chr(10) ||
' splines=ortho' || chr(10) ||
' size=8.25' || chr(10) ||
' label="Referential Integrity around &&TABLE_NAME table.";' || chr(10) ||
' rankdir=LR;' || chr(10) ||
' edge \[color=blue4, arrowhead=crow\];' || chr(10) || chr(10) ||
' // &&TABLE_NAME is the starting table.' || chr(10) ||
' "&&TABLE_NAME" \[shape=box, style=filled, color=blue4 fillcolor=cornflowerblue\];' || chr(10) || chr(10) ||
' // The remaining nodes are this style.' || chr(10) ||
' node \[shape=box, style=filled, color=dodgerblue2, fillcolor=aliceblue\];' || chr(10) || chr(10) ||
' // These are the parent -> child edges.' || chr(10)
from dual;

with refint as (
    select constraint_name, table_name, r_constraint_name
    from dba_constraints
    where constraint_type = 'R'
    and owner = upper('&&owner_name')
),
primekey as (
    select constraint_name, table_name
    from dba_constraints
    where constraint_type in ( 'P', 'U')
    and owner = upper('&&owner_name')
),
links (child_table, f_key, parent_table, p_key) as (
    select refint.table_name, refint.constraint_name, primekey.table_name, refint.r_constraint_name
    from primekey join refint on primekey.constraint_name = refint.r_constraint_name
)
select distinct chr(9) || '"' || links.parent_table || '" -> "' || links.child_table || '";' as dot
--select distinct level, links.\*
from links
start with links.parent_table = upper('&&TABLE_NAME')
connect by nocycle links.child_table = prior links.parent_table
order by 1
--order by parent_table, child_table, f_key
;

select '}' from dual;

spool off
```

The output looks remarkably similar to the following example:

```
// dot -Tpdf -o this_file.pdf this_file.dot

digraph refInt {
    splines=ortho
    size=8.25
    label="Referential Integrity around ORDER_ITEM table.";
    rankdir=LR;
    edge \[color=blue4, arrowhead=crow\];

    // ORDER_ITEM is the starting table.
    "ORDER_ITEM" \[shape=box, style=filled, color=blue4 fillcolor=cornflowerblue\];

    // The remaining nodes are this style.
    node \[shape=box, style=filled, color=dodgerblue2, fillcolor=aliceblue\];

    // These are the parent -> child edges.
    "ADDRRESS" -> "CUSTOMER";
    "COLLECTION_METHOD" -> "ORDER_ITEM";
    "COMPENSATION_LEVEL" -> "ORDER_ITEM";
    "CUSTOMER" -> "CUSTOMER_ORDER";
    "CUSTOMER" -> "ORDER_ITEM";
    "CUSTOMER_GRP" -> "CUSTOMER";
    "CUSTOMER_GRP" -> "PRICING_BAND";
    "CUSTOMER_ORDER" -> "DISCOUNT_USAGE";
    "CUSTOMER_ORDER" -> "ORDER_ITEM";
    "DISCOUNT_PLAN" -> "DISCOUNT_USAGE";
    "DISCOUNT_USAGE" -> "CUSTOMER";
    "ORDER_ORIGIN" -> "ORDER_ITEM";
    "ORDER_ITEM" -> "EXTRAS";
    "ORDER_ITEM" -> "ADJUSTMENTS";
    "ORDER_ITEM" -> "ORDER_ITEM";
    "ORDER_ITEM" -> "REFUND_PAYMENTS";
    "ORDER_ITEM_STAT" -> "ORDER_ITEM";
    "ORDER_STATUS" -> "CUSTOMER_ORDER";
    "PRICING_BAND" -> "WEIGHT_RANGE";
    "WEIGHT_RANGE" -> "ORDER_ITEM";
}
```

### Generating the Image

The first line of the output file, `refint.dot`, shows the command line required to create a PDF version of the ERD. You can specify PNG, JPG, etc as desired. SVG is good for images that need to be scalable (and is usually the better quality output). To generate an SVG image, run the following command line:

```bash
dot -Tsvg -o refint.svg refint.dot
```

A file by the name of `refint.svg` will be created. And it looks like the following, for this particular example.

![Sample Referential Integrity Diagram](/images/refint.dot_.png "Sample Referential Integrity Diagram")


### Caveats

The diagram is only as good as the referential integrity in your target database. This much should be obvious - if there are no referential integrity constraints, then all you will get is a single entity. If that's the case, I'd be looking for another job as I suspect that all the required checking is being done in the application, rather than in the database - best avoided!

And finally, you will not generate a full schema ERD with this code, but it's handy for stuff around and about a particular table.

Enjoy.
