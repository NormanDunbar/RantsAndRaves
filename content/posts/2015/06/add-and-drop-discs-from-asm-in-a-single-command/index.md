---
title: "Add and Drop Discs From ASM in a Single Command"
date: "2015-06-27"
categories: 
  - "oracle"
---

Recently I was tasked to do something that I hadn't done before. I was required to swap out all the existing discs in the two diskgroups +DATA and +FRA, with minimal downtime. Almost all the places I looked seemed to indicate that I had to add the new discs, rebalance, drop the old discs and rebalance again. My colleague, Ian Greenwood, had a much better idea - thanks Ian.

```sql
alter diskgroup DATA add disk
--
'/path/to/disk_1' name DISK_1001,
'/path/to/disk_2' name DISK_1002,
...
'/path/to/disk_n' name DISK_100N
--
drop disk
--
DISK_0001,
DISK_0002,
...
DISK_000N
--
-- This is Spinal Tap!
--
rebalance power 11;
```

Then the same again for +FRA and we were done. Well, I say done, once the rebalance had finished we were done, and the Unix team could then remove completely, the old discs. That did need ASM to be bounced though, which was a bit of a nuisance for the (one) database on the server, but the users were happy to let us take it down.

Job done and very little messing around. Sometimes, it's helpful to look at the Oracle Manuals before hitting MOS or Google (other web search engines are available - but they are not as good!) for hints when you have new stuff to do.

Yes, I spell disc with a 'c' while Oracle spell it with a 'k'. :-)
