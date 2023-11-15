---
title: "Useful AIX Unix Commands"
date: "2018-09-07"
categories: 
  - "linux"
  - "unix-other"
---

#### proctree

The `proctree` command acts like the Linux `pstree` and displays the hierarchy of processes from a given starting process id.

#### View Large Files

If, when you `view` a large file you get an errors about it being too big, try the following:

```
echo "set ll=3501720 dir=/tmp" >> ~/.exrc
```

That will allow you to read up to 3501720 lines in a single text file.
