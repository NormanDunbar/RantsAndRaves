---
title: "Rsync - To Slash or Not To Slash?"
date: "2013-02-11"
categories: 
  - "linux"
---

Rsync is great for making sure that a destination directory is synchronised with a source directory. However, do you add a slash to the source and/or destination directory names, or do you not?

The answer is, it depends. Without a slash on the source directory means _copy both the source directory, and the contents (recursively if specified) to the destination directory_ while adding a trailing slash means _only copy the contents of the source directory, recursively if specified, to the destination_. Easy?

If we take the following as the source directory:

```bash
$ tree testing
```
```text
testing
|-- another
|   +-- wilma
+-- betty
+-- fred
+-- nested
    +-- barney
```

The destination is an empty directory named test_backup.

## No Slashes

The first test has no slashes on any of the directories.

```bash
$ rm -r test_backup/*
$ rsync --archive --recursive testing test_backup
$ tree test_backup
```
```text
test_backup
+--testing
   +-- another
   |   +-- wilma
   +-- betty
   +-- fred
   +-- nested
       +-- barney
```

You can see that the whole hierarchy of the testing directory has been recreated _within_ the destination directory.

## Slash on Destination Only

The second test, after clearing out the destination directory, adds a slash to the end of the destination directory.

```bash
$ rm -r test_backup/*
$ rsync --archive --recursive testing test_backup/
$ tree test_backup
```
```text
test_backup
+--testing
   +-- another
   |   +-- wilma
   +-- betty
   +-- fred
   +-- nested
       +-- barney
```

So, there's no difference there. A slash on the destination directory appears to have no effect.

## Slash on Source Only

```bash
$ rm -r test_backup/*
$ rsync --archive --recursive testing/ test_backup
$ tree test_backup
```
```text
test_backup
+-- another
|   +-- wilma
+-- betty
+-- fred
+-- nested
    +-- barney
```

This is different. The _contents_ of the source directory have been duplicated into the destination directory.

## Slashes on Both

```bash
$ rm -r test_backup/*
$ rsync --archive --recursive testing/ test_backup/
$ tree test_backup
```
```text
test_backup
+-- another
|   +-- wilma
+-- betty
+-- fred
+-- nested
    +-- barney
```
And this one again shows that having a slash on the destination directory has no effect.
