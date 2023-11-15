---
title: "Build 64bit OCILIB Libraries for CodeBlocks"
date: "2019-02-15"
categories: 
  - "oracle"
  - "windows"
---

I tend to compile with gcc, in a bash session, on Windows 7. I use Code::Blocks as my IDE of choice and one of my projects, well, quite a few, use the excellent OCILIB library for accessing Oracle databases, by Vincent Rogier. I can't recommend this library highly enough.

However, it comes with a Code::Blocks project file to build 32 bit libraries, but I need 64 bits. Here's how I do it.

None of the following is needed if you use something like Visual C as your compiler as OCILIB supplied 32 and 64 bit libraries for Visual C/C++. I don't use Visual C/C+ so I need to build my own libraries!

### Download the source code

Got to [https://github.com/vrogier/ocilib/releases](https://github.com/vrogier/ocilib/releases) and grab the latest release. Watch out for version 4.6.0 as it will not compile on a system with an Oracle 12.1 client - but I have a fix. Version 4.6.1 (when it's available) has the fix built in.

You need the file `ocilib-x.x.x-windows.zip`, so download it to wherever you keep your source files, and unzip it.

### Amend the Supplied Project File

There is a project file called `proj\mingw\ocilib_static_lib_mingw.cbp`, so open that in Code::Blocks. You will notice that it has two options - 'Release - ANSI' and 'Release - UNICODE'. I use the ANSI version, but the following gives instructions for both.

The project as supplied uses the built in compiler which is a 32 bit only, version of `gcc`. My system has a separately installed version that compiles 32 and 64 bit applications. So I need to change the existing targets to use my compiler instead of the built in one.

### Change the Default Compiler

Go to Project -> Build Options.

Select the top level `ocilib_static_lib_mingw` option on the left. Do not select either of the two release targets at this stage.

The default, built in 32 bit, compiler is named 'GNU GCC Compiler' I need to change this to 'GNU GCC Compiler 32/64bit (TDM-GCC-64)' which is the name my separately installed compiler is called. If you are prompted to save the settings, choose 'Yes' then 'OK' on all the remaining dialogues that pop-up. Finally, 'OK' your way back to the main screen.

The compiler has now been changed for all targets currently defined. Now, to add the new 64 bit targets.

### Add 64bit Targets.

Go to Project -> Properties and select the 'Build targets' tab.

Select 'Release - ANSI' on the left and click the 'Duplicate' button, change the name to 'Release 64bit - ANSI' and 'OK'.

Select the new target, if not already selected.

Under the 'Output filename' option, towards the right side, change the 'lib32' to 'lib64'. Everything else remains the same.

Now repeat the above to create the 'Release 64bit - UNICODE' target, and 'OK' back to the main screen.

That now gives us 4 targets, but they are all 32 bit at present. We now need to change the compiler options.

### Change the Compiler Options

Go to Project -> Build Options and select the very top of the tree - `ocilib_static_lib_mingw`. Do not select any of the sub-targets, yet.

On the 'Compiler settings' tab, scroll down the options list and make sure that both 'Target X86 (32bit)' and 'Target X86 (64bit)' are unselected.

Click 'Release - ANSI' on the left, and select 'Target X86 (32bit)'.

Click 'Release - UNICODE' on the left, and select 'Target X86 (32bit)'. Select 'yes' if prompted to save changes.

Click 'Release 64bit - ANSI' on the left, and select 'Target X86 (64bit)'. Select 'yes' if prompted to save changes.

Click 'Release 64bit - UNICODE' on the left, and select 'Target X86 (64bit)'. Select 'yes' if prompted to save changes.

Now 'OK' back to the main screen.

### Build the Libraries

Now that we have 4 targets, it's time to build the different libraries. In the main screen, select each of the 4 targets in turn, and then select Build -> Rebuild to make sure that any existing versions are rebuilt with the new compiler and options.

When done, there should be two new files in each of the `lib32` and `lib64` folders, these will be named `libociliba.a` and `libocilibw.a`.

### Fix 4.6.0 Errors

If you get an error about `OCI_ATTR_COL_PROPERTY_IS_CONID` then you need to apply the following fix if release 4.6.1 is not yet available.

Edit the file `src/column.c` and change the code to the following at lines 260 onwards - there's only two lines to add.

```
  259             if (value & OCI_ATTR_COL_PROPERTY_IS_GEN_BY_DEF_ON_NULL)  
  260             {  
  261                 col->props |=  OCI_CPF_IS_GEN_BY_DEFAULT_ON_NULL;  
  262             }  
  263  
  264 #if OCI_VERSION_COMPILE >= OCI_18_1  
  265  
  266             if (value & OCI_ATTR_COL_PROPERTY_IS_LPART)  
  267             {  
  268                 col->props |= OCI_CPF_IS_LPART;  
  269             }  
  270  
  271             if (value & OCI_ATTR_COL_PROPERTY_IS_CONID)  
  272             {  
  273                 col->props |= OCI_CPF_IS_CONID;  
  274             }  
  275 #endif  
  276  
  277         }  
  278     }
```

All that is required is to add the two lines numbered 264 and 275 above, but _not_ the line numbers please! You should now be able to compile the code.

Job done, I can now build 32 and 64 bit versions of my applications to access the databases.

### Undefined References?

If, when compiling an _application_ that uses OCILIB, you see lots of error messages stating something like:

Undefined reference to 'OCI_Initialize'
Undefined reference to 'OCI_Cleanup'
Undefined reference to 'OCI_ ...'

Then you have a wee bit of a problem. Although the 32 and 64 bit OCILIB compilations have obvioulsy worked ok, your Oracle Client software is the wrong bit size for the application's bitness.

I've had this problem where a system happily compiled OCILIB in 32 and 64 bit mode, but when I came to compile an application in 64 and 32 bit mode, it failed on the latter as my Oracle Client is only 64 bits. Bummer - I can't compile 32 bit applications on that particular system.

To get around this, you must compile the application (and, maybe also OCILIB) with the correct bit sized Oracle Client on the path
