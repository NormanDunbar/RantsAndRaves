---
title: "Need to Compile a 32 bit Application on a 64 bit Linux System?"
date: "2013-05-13"
categories: 
  - "linux"
---

I have a small utility, written in C, that I need to use from time to time. I wrote it back in 2009 when 32 bit systems were all the rage. It reads a certain type of data file and outputs lots of, ahem, _interesting_ information about the contents. When I compiled it on my 64 bit laptop, it produced complete and utter rubbish! How did I fix it?

The data file that the utility reads is a [Firebird database](http://firebirdsql.org "Firebird Databases") file which has a number of `long` fields in various `struct`s. On a 32 bit Linux system, a `long` is the same size as an `int` - both are 32 bit.

Thanks to Vlad Khorsun, one of the Firebird Development team (of volunteers) I realised that under a 64 bit system, `int`s remained as 32 bit while `long`s got a wee bit bigger at 64 bits - hence my `struct`s were complete nonsense on a 64 bit system.

There are two solutions, well, at least two:

- Rewrite the code to take account of the world of 32 and 64 bit development systems. This would involve discovering, somehow, if the utility was being built on a 32 or 64 bit system, and changing the definition of a long variable or struct member accordingly.
- Compile the system and make it believe it's running on a 32 bit system. No code changes required!

Being lazy, and because there's only a slim possibility of this code being released into the wild, I chose the latter option. It turned out to be quite simple.

First, I needed to install the gcc multilib support packages on my system to support compiling 32 bit C code:

```bash
sudo apt-get install gcc-multilib
```

As I will also need to perform 32 bit C++ code compilations, I also need to install the g++ multilib support packages:

```bash
sudo apt-get install g++-multilib
```

Now, when I compile the code, I use the `-m32` option to gcc, as follows:

```bash
gcc -m32 -o fbdump32 fbdump.c
```

Simple, and it works. However, the correct solution would be to correctly define my `struct`s with proper C/C++ standard data types that guarantee that the width will always be 32 bits and/or determine whether I'm compiling on 32 or 64 bit systems. For example:

**Data types that guarantee specific widths**

- [u]int8_t are 8 bit.
- [u]int16_t are 16 bit.
- [u]int32_t are 32 bit.
- [u]int64_t are 64 bit.

   Maximums and minimums for these data types are defined as:

- [U]INT8_MIN and [U]INT8_MAX
- [U]INT16_MIN and [U]INT16_MAX
- [U]INT32_MIN and [U]INT32_MAX
- [U]INT64_MIN and [U]INT64_MAX

**What bitsize am I compiling on**

```C
// Am I compiling on a 64 bit system by any chance? 
#if __WORDSIZE==64
... // Do 64 bit stuff here.
#else
... // Do 32 bit stuff here.
#endif
```

Cheers.  \
Norm.
