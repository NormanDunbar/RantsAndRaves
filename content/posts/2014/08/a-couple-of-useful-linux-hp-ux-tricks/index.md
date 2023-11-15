---
title: "A Couple of Useful Linux & HP-UX Tricks"
date: "2014-08-23"
categories: 
  - "linux"
---

Recently at work, I was on an HP server and needed to `grep -B3` a large log file to find the three lines prior to a number of Oracle error messages I was searching for, in order to fix things. It turned out that HP-UX doesn't have the -B (or the -A) options. Bummer. A bit of `awk` fixed the former, but the latter I leave as an _exercise for the reader_ as they say!

```awk
awk '/^ORA-/ {for (i = 1; i <= x;) print a[i++]; print; print "---"} {for (i = 1; i < x; i++) a[i] = a[i + 1]; a[x] = $0;}'  x=3 file_name
```

In my case, setting `x=3` allowed me to display the three lines prior to the error line which itself began with `ORA-`.

In use, do this:

- Change the search term between '/' in the above, to look for your text.
- Set x to however many lines _before_ the found text you wish to display.
- Pass the file name to be searched as the last parameter of the command.

That's it. Simple, as they say! You can easily build the above into a shell script and simply pass it the three parameters for x, the search text and the file name. Saves getting all those punctuation characters in the right place!

### Dos2Unix for Servers That Don't Have it Installed

`Dos2Unix` is very handy when you have to use a secure file transfer system - `sftp, scp` etc - to send text files from Windows to a Unix server. Because the servers are securely locked down, unlike your home PC, you cannot simply install packages as and when you like.

The problem with the secure protocols is that they send in binary, so Windows text files get transferred with the CR/LF end of line characters intact, which is irritating when you subsequently try to edit the file in `vi` (`emacs` is also not installed, thankfully!)

To fix the problem, you could edit the file manually and remove the visible `^M` characters from each line. You _could_ but why would you, `sed` and `vi` allow you to do it in one command:

```text
:1,$s/^M//g
```

Of course, if you simply type in the two characters, ^M, shown above, nothing will be replaced. However if you enter them as:

CTRL-v CTRL-m

Where the above means press the CTRL key and while holding it down, press the v (or m) key.

Now it works!
