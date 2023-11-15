---
title: "Trace Collier Updated Again"
date: "2017-01-13"
categories: 
  - "oracle"
---

Trace Collier has been updated again. Mostly bug fixes, but there's a little enhancement too. The current release is 0.21.

Trace Collier is a utility that parses an Oracle trace file, with binds listed (event 10046 level 4 or 12, etc) and extracts all the user submitted SQL statements and writes them to an output file with the bind variables replaced by the actual literals used when the statement was executed.

You can download the C source code from my GitHub repository [https://github.com/NormanDunbar/TraceMiner](https://github.com/NormanDunbar/TraceMiner), or download it directly from [https://github.com/NormanDunbar/TraceMiner/archive/master.zip](https://github.com/NormanDunbar/TraceMiner/archive/master.zip) and compile it for your own machine. Currently, I know that the utility is _happily_ running on Windows, AIX and Linux.

A number of fixes have been made between version 0.19 and the current, 0.21:

V0.20, which didn't get released, made the following changes:

- Data type 96, `NCHAR` or `NVARCHAR`, intermittent bug fixed. Sometimes it tried to extract the hex values from the _previous_ line of text, not the current one. Weird! Thankfully, verbose output showed it up.
- `MAXBINDS` upped from 50 to 150 - just because!
- `CLOSE` of a cursor now gets handled by removing it from the linked list.

V0.21, which is the current release, has these changes:

- `CLOSE` of a cursor is now handled differently. A large trace file showed that Oracle can re-parse the same SQL after a cursor has been `CLOSE`d and only if the cursor id was never used with a different SQL statement. Instead of `PARSING IN CURSOR #1234 …` followed by the full SQL text, it simply does `PARSE #1234` again with no SQL text! That caused the subsequent `EXEC #1234` to abort the program as the cursor wasn't in the linked list - because `CLOSE #1234` removed it - and was considered a fatal problem.
- Data Type 96, `NCHAR` or `NVARCHAR` can cause problems as it is possible to output more than one line of hex values. Up until now, I've only ever seen one line but this is now fixed. The hex values in the second and subsequent lines are simply ignored. ;-)
- There was a segfault at EOF, _only_ if running in verbose mode and with config.h having set with `OFFSETFORRICH = 0`. It was when freeing the memory allocated for the SQL statement.
- The verbose output has been tidied up a bit. Also, if you ever need to extend the `MAXBINDS` or `MAXBINDSIZE`, you get a message in the output file, not in the debug file. Just in case you don’t have a debug file!

Enjoy.
