---
title: "What's wrong with this file system?"
date: "2009-05-11"
categories: 
  - "qdosmsq"
---

On the QL we have traditionally had a 36 character limit on file names after the initial (5 character) device name. Something like this:

flp1\_source\_code\_myfile\_c

In the above, _flp1\__ is the device name and is not included in the 36 character limit. There can be a network device tagged on the front as well, and this too is not included in the filename limit.

Technically, in the above, _source_ and _code_ are both directories (the default directory separator is the underscore character) while _myfile\_c_ is the filename and extension.

In the original days of the QL, all we had was a pair of built in [microdrives](http://terdina.net/ql/mdv.html "QL Microdrives") and so the file name limit wasn't too much of a hassle because we only had about 100KB (yes, KB) to play with and there were no actual directories as such.

Then we moved on to floppies and even a 40 MB (yes, MB) hard drive - and suddenly, this limit began to look rather silly, especially as we now had proper directories.

The problem, as I see it is simple. Inside the directory entry for a filename, a space of 36 characters has been reserved. Unfortunately, given the above filename as an example, the following will happen:

- The root directory on the floppy disc will contain an entry for a directory named _source_.
- The _source_ directory will have an entry for a directory named _source\_code_.
- The _source\_code_ directory will have an entry for a file named _source\_code\_myfile\_c_.

You can see the problem, the entire directory structure, less the device name, is replicated all the way down the tree. Surely it makes sense, even with a 36 character limit, to only have the appropriate parts of the full path in each directory. For example:

- The root directory on the floppy disc will contain an entry for a directory named _source_.
- The _source_ directory will have an entry for a directory named _code_.
- The _code_ directory will have an entry for a file named _myfile\_c_.

With this system, we would be able to have almost unlimited depth to our disc structures - possibly not wise - and each part of a file's full path would be limited to 36 characters, not the entire path itself.

Well, I think it makes sense anyway.

Cheers.
