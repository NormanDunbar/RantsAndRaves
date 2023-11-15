---
title: "Transportable Tablespace Migrations with Expdp/Impdp"
date: "2013-02-21"
categories: 
  - "oracle"
  - "rants-raves"
---

In the old days of `exp`/`imp` doing a Transportable Tablespace export/import was relatively simple - unless you had Spatial data, in which case, it wasn't. Then we got hold of `expdp`/`impdp` and it became "_different_". It now seems that in order to do a Transportable Tablespace import with `impdp`, you _don't_ tell it to do one! Confused?

In the old days, you told both `exp` and `imp` which tablespaces you were transporting using the `TRANSPORT_TABLESPACE` and `TABLESPACES` parameters to `exp`, and the same with `imp`. Life was good and symmetrical back then!

With `expdp`, we have a similar parameter whereby we simply list the tablespaces we wish to transport in the `TRANSPORT_TABLESPACES` (note, plural) parameter.

You would think that a similar arrangement would exist with `impdp` wouldn't you? Well it does. Sort of!

There is indeed a `TRANSPORT_TABLESPACES` (note, plural) parameter on `impdp` but, if you use it, you _must_ also specify a database link name in the `NETWORK_LINK` parameter _and_ that link must exist in the importing database _and_ it must point back to the exporting database. There is no intermediate dump file, the metadata is unloaded from the source database over the database link. Smart?

Not quite. Imagine that you have a production database to export and then import into a QA database, for example. However, the two are on separate networks, and there's an air gap between the two. You cannot set up a database link from the QA database to the Production one to get at the metadata for the Transportable Tablespace import.

If you try, without a database link in place and working, `impdp` will simply barf.

The solution is hidden away in a note attached to the `TRANSPORT_TABLESPACES` in the **Utilities** manual, `impdp` section, in the locked filing cabinet, in the office in the basement, behind the leopard guarding the stairs, down the dark corridor etc. (Douglas Adams.)

What you have to do is _not_ use the `TRANSPORT_TABLESPACES` parameter as you would imagine, instead, you list the transported data files using the `TRANSPORT_DATAFILES` and avoid the `TRANSPORT_TABLESPACES` like the plague! Goodbye symmetry, it was nice of you to drop in!

So far, so bad. However, my [original problem with deferred segment creation](/posts/2012/11/oracles-deferred-segment-allocation-breaks-transportable-tablespace-imports/ "Oracle’s Deferred Segment Allocation Breaks Transportable Tablespace Imports.") still exists even with `impdp` instead of `imp`.

```bash
$ impdp directory=... dumpfile=... logfile=... transport_datafiles=file1.dbf ...
```
```text
...
Processing object type TRANSPORTABLE_EXPORT/TABLE
ORA-39083: Object type TABLE:"NORMAN"."TEST_TABLE" failed to create with error:
ORA-01647: tablespace 'TEST_TABLESPACE' is read-only, cannot allocate space in it
...
```

Of course it's read-only, it doesn't exist in the database because `impdp` is supposed to create it, load up the metadata and connect that with the data files etc. Then I have to login and make the tablespace read write when I'm done.

Sigh!
