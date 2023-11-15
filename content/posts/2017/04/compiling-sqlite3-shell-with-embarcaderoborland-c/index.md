---
title: "Compiling Sqlite3 Shell with Embarcadero/Borland C."
date: "2017-04-26"
categories: 
  - "sqlite"
  - "windows"
---

I wanted to compile the [sqlite3](https://sqlite.org) shell on Windows, using my [Free Embarcadero C compiler](https://www.embarcadero.com/free-tools), but it didn't work. It was quite easy to fix, but if you are affected, read on.

### First Attempt

After unzipping the _amalgamated_ source files a default compilation was attempted with the following command line. The `-tCM` option simply says to create a console based non-windows application.

```
bcc32c -o sqlite3.exe shell.c sqlite3.c -tCM
```

Which results in the following compiler warnings and linker errors:

```
shell.c:158:3: warning: implicit declaration of function '_setmode' is invalid in C99...
shell.c:7018:26: warning: implicit declaration of function '_isatty' is invalid in C99...
...
Error: Unresolved external '__isatty' referenced from C:\\SQLITE3\\SHELL-264807.O
Error: Unresolved external '__setmode' referenced from C:\\SQLITE3\\SHELL-264807.O
```

The compiler warnings gave me the impression that something called `_isatty` and `_setmode` do not exist, or something is not setting up their function definitions correctly. That's usually the case with the warnings shown anyway.

Because all the warnings are in the file `shell.c`, that's what we need to look at.

### Fixing the shell.c Source File

Open `shell.c` in your favourite text editor. Alternatively, use Notepad!

Search for `_isatty`. You should find it around line 104.

Replace this code:

```C
...
# define isatty(h) _isatty(h)
...
```

With the following:

```C
...
# if !defined (__BORLANDC__)
# define isatty(h) _isatty(h)
# endif
...
```


Embarcadero aka Borland C doesn't have anything named `_isatty` but it does have `isatty` and it _is_ the function we need to be calling. Moving on...

Search for _setmode. You should find it around line 158.

Replace this code:

```C
...
_setmode(_fileno(file), _O_BINARY);
...
_setmode(_fileno(file), _O_TEXT);
...
```

With the following:

```C
...
#if defined (__BORLANDC__)
setmode(_fileno(file), _O_BINARY);
#else
_setmode(_fileno(file), _O_BINARY);
#endif
...
#if defined (__BORLANDC__)
setmode(_fileno(file), _O_TEXT);
#else
_setmode(_fileno(file), _O_TEXT);
#endif
...
```

Again, Embarcadero C doesn't have `_setmode` as it has `setmode` instead, so that's what we need to be calling here, twice.

Once the changes have been saved, exit from the editor and recompile.

### Second Attempt

The same command line is used:

```
bcc32c -o sqlite3.exe shell.c sqlite3.c -tCM
```

Which results in the following:

```
Embarcadero C++ 7.20 for Win32 Copyright (c) 2012-2016 Embarcadero Technologies, Inc.
shell.c:
sqlite3.c:
Turbo Incremental Link 6.75 Copyright (c) 1997-2016 Embarcadero Technologies, Inc.
```

No errors, no warnings. Job done, we have a shell!

### Testing, Testing

It. Just. Works!

Databases can be created, deleted, opened closed, attached and so on, tables and indexes etc can be created and used. Life is good!
