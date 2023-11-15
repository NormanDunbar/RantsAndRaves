---
title: "Trace Collier and TraceAdjust"
date: "2017-03-20"
categories: 
  - "oracle"
---

**Trace Collier** has been updated and rewritten. Numerous bugs and foibles have been fixed.

**TraceAdjust** is a new, useful, utility to carry out some pre-processing on a trace file before you have to use your own weary eyes to work thorough potential Oracle performance problems! That's why I wrote it!

### Trace Collier

My old utility has been totally rewritten in C++ rather than vanilla C, which has had the bonus of allowing me to fix some bugs, and do away with the need to recompile whenever you hit a system limit of some kind. You can view the readme.pdf file [here](https://github.com/NormanDunbar/TraceMiner2/blob/master/README.pdf).

Source code is available from [GitHub](https://github.com/NormanDunbar/TraceMiner2) as you will need to compile this utility to match your database server, or analysis systems. Tested on Windows and Linux.

### TraceAdjust

TraceAdjust is a new utility which massages a tracefile to do the following:

- Add a decimal point in the tim values, to separate full seconds from micro-seconds. My eyes are too old to do it manually!
- Add a _delta_ to the end of the trace line to show the difference between the previous tim and the current tim. This saves me doing it with a calculator, and multi-digit tim values!
- Adds a running delta since the most recent timestamp record in the trace file. Perhaps not as useful as the above, but it works for me.
- Convert the deltas so far, into actual, locally oriented timestamps showing resolution down to the micro-second.

Having the tracefile show these values already worked out does make life a bit easier when you have to get dirty in the raw traces.

Have a look at the Readme.pdf file [here](https://github.com/NormanDunbar/TraceAdjust/blob/master/README.pdf), and, download the source code which, as ever, is available from [GitHub](https://github.com/NormanDunbar/TraceAdjust) as you will need to compile this utility to match your database server, or analysis system. Tested on Windows and Linux.
