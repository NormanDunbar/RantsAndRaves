---
title: "Clone Oracle Home Easily"
date: "2012-02-09"
categories: 
  - "oracle"
---

[Frits Hoogland's post](http://fritshoogland.wordpress.com/2010/07/03/cloning-your-oracle-database-software-installation/ "http://fritshoogland.wordpress.com/2010/07/03/cloning-your-oracle-database-software-installation/") on how to clone your Oracle Home rather than installing from scratch. Nice!

One thing to beware of, if you have already applied a PSU, then you must ensure that you also include the `.patch_contents` hidden directory, if you don't, you will not be able to apply any further PSUs to the cloned home(s) created from the tarball.
