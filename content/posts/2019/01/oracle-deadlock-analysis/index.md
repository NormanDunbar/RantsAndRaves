---
title: "Oracle Deadlock Analysis"
date: "2019-01-17"
categories: 
  - "oracle"
---

There's a new utility to assist in diagnosing the underlying cause of Oracle deadlocks. Interested?

You can download source code as well as binaries for Linux and Windows at [https://github.com/NormanDunbar/DeadlockAnalysys/releases](https://github.com/NormanDunbar/DeadlockAnalysys/releases).

All you have to do is execute the DeadlockAnalysis utility with a list of Oracle trace files on the command line. Each trace file will get its own report, in HTML format, in the same directory as the trace file.

The report files use a CSS style sheet to format the output. This file can be edited to use your own installation style, if necessary, and if the style sheet exists in the output directory when the utility is executed, it will not be overwritten.

That's about it really! There's not much more to say except, enjoy! (Ok, one last thing, yes, I _do_ wish I had managed to spell "analysis" correctly when setting up the github repository. Sigh.)
