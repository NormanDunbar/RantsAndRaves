---
title: "Docbook - Creating Indexes"
date: "2012-01-13"
categories: 
  - "docbook"
---

I've been doing a bit of Docbook work recently - converting a paper based manual into a Docbook one that can then be used to generate all kinds of different output from the same input file. Very useful.

I needed to create an index that correctly reflected the contents of the new format rather than simply copying the old one - where the pages would no longer have matched up.

I found a couple of good articles, namely:

[http://www.xml.com/pub/a/2004/07/14/dbndx.html](http://www.xml.com/pub/a/2004/07/14/dbndx.html "http://www.xml.com/pub/a/2004/07/14/dbndx.html")

and, of course, this one:

[http://www.sagehill.net/docbookxsl/GenerateIndex.html](http://www.sagehill.net/docbookxsl/GenerateIndex.html "http://www.sagehill.net/docbookxsl/GenerateIndex.html")

**UPDATE**: Of all the people you would expect to get it right, Bob Stayton is the one. Except he got it wrong. In the link above to [www.sagehill.net](http://www.sagehill.net "www.sagehill.net"), there is an example thus:

```xml
<indexterm class="startofrange" id="makestuff"\>
   <primary>Makefiles</primary>
</indexterm>
 ... 
<indexterm class="endofrange" startref="makestuff"\>
   <primary>Makefiles</primary>
</indexterm>
```

It should be as follows _without_ the `<primary>` on the `endofrange` indexterm. 

```xml
<indexterm class="startofrange" id="makestuff"\>
   <primary>Makefiles</primary>
</indexterm>
 ... 
<indexterm class="endofrange" startref="makestuff"\>
</indexterm>
```

If you do it with a primary, you get the page range as expected, but with the final page duplicated, as in:

```text
Subject matter 123-127, 127
```

Remove the primary and it just works.
