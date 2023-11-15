---
title: "Tnsnames.ora Parser"
date: "2014-08-23"
categories: 
  - "oracle"
---

Have you ever wanted to use a tool to parse the manually typed up "stuff" that lives in a tnsnames.ora file, to be absolutely certain that it is correct? Ever wanted some tool to count all the opening and closing brackets match? I may just have the very thing for you.

Download the binary file [Tnsnames.Parser.zip](http://qdosmsq.dunbar-it.co.uk/downloads/TnsnamesParser/ "Tnsnames.Parser.zip") and unzip it. [Source code](https://github.com/NormanDunbar/grammars-v4/tree/master/tnsnames "Source Code on Github") is also available on Github.

When unzipped, you will see the following files:

- `README` - this should be obvious!
- `tnsnames_checker.sh` - Unix script to run the utility.
- `tnsnames_checker.cmd` - Windows batch file to run the utility.
- `antlr-4.4-complete.jar` - Parser support file.
- `tnsnames_checker.jar` - Parser file.
- `tnsnames.test.ora` - a valid tnsnames.ora to test the utility with.

The README file is your best friend!

All the utility does is scan the supplied input file, passed via standard in, and writes any syntax or semantic problems out to standard error.

### Working Example

There are no errors in the tnsnames.test.ora file, so the output looks like the following:

```bash
./tnsnames_checker.sh < tnsnames.test.ora

Tnsnames Checker.
Using grammar defined in tnsnames.g4.
Parsing ....
Done.
```

### Non-Working Example

After a bit of fiddling, there are now some errors in the tnsnames.test.ora file, so the output looks like the following:

```bash
./tnsnames_checker.sh < tnsnames.test.ora

Tnsnames Checker.
Using grammar defined in tnsnames.g4.
Parsing ....
line 5:12 missing ')' at '('
line 8:16 extraneous input 'address' expecting {'(', ')'}
Done.
```

You can figure out where and what went wrong from the messages produced.

Have fun.
