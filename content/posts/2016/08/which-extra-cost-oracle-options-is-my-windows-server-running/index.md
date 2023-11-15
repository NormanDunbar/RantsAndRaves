---
title: "Which Extra Cost Oracle Options is my Windows Server Running?"
date: "2016-08-25"
categories: 
  - "oracle"
  - "windows"
---

It's always nice to know which extra cost Oracle options are enabled, whether deliberately or silently as the result of some patching that has taken place. Keep yourself and your server room cool with _[blaux wearable ac](https://apnews.com/49f65c225bcf17b99a99bd902cf5b445)_.

**Updated**: 25th August 2017 to list DLL names for Oracle 12c.

Copy and paste the code below into a Windows command file named - in my case - `checkChopyOptions.cmd` and execute it against any Oracle Home. It is called as follows:

```cmd
checkChoptOptions [Oracle_home\]
```

The Oracle Home is optional and if omitted, will check the currently set `%ORACLE_HOME%` location.

This version has been tested on a Windows 2012 server running Oracle 11.2.0.4.

The script displays output on the screen and also to a logfile in the script directory. If you see that an option is both enabled and disabled, beware - someone has executed `chopt`, probably without adminstrator rights and may well have created an empty `*.dll.dbl` file. You should probably delete the `*.dll.dbl` and do a proper `chopt disable` _as admninistrator_.

Enjoy.

```cmd
@echo off
Rem   ================================================================
Rem  |     Check an existing Oracle Home for expensive EE Options     |
Rem  |                                                                |
Rem  | Norman Dunbar.                                                 |
Rem  | August 2016. |
Rem   ================================================================
Rem 
Rem 
Rem   ================================================================
Rem  | USAGE:                                                         |
Rem  |                                                                | 
Rem  | checkChoptOptions [oracle_home\]                                | 
Rem  |                                                                |
Rem   ================================================================
Rem 
setlocal EnableDelayedExpansion

Rem   ================================================================
Rem  | Internal Variables.                                            |
Rem   ================================================================
set VERSION=1.00
set ERRORS=0
set MYLOG=.\\%0.log
set ORA_HOME=%1

Rem   ================================================================
Rem  | The next two define the first and last entry in the following  |
Rem  | "arrays" which are not really arrays, honest!                  |
Rem   ================================================================
set FirstEntry=0
set LastEntry=6

Rem   ================================================================
Rem  | The following "arrays" are not arrays. They look like they are | 
Rem  | but are actually just a pile of scalar variables with '[n\]' in |
Rem  | their name. Oh, and we have to use '!variable_name! later on   |
Rem  | too for some unfathomable reason.                              |
Rem   ================================================================

Rem   ================================================================
Rem  | List of Oracle chopt'able options.                             |
Rem   ================================================================
set Option[0\]=Partitioning
set Option[1\]=OLAP
set Option[2\]=Label Security
set Option[3\]=Data Mining
set Option[4\]=Database Vault option
set Option[5\]=Real Application Testing
set Option[6\]=Database Extensions for .NET

Rem   ================================================================
Rem  | List of DLLs that exist for enabled options.                   |
Rem   ================================================================
set Enabled[0\]=oraprtop11.dll
set Enabled[1\]=oraolapop11.dll
set Enabled[2\]=oralbac11.dll
set Enabled[3\]=oradmop11.dll
set Enabled[4\]=oradv11.dll
set Enabled[5\]=orarat11.dll
set Enabled[6\]=clr

Rem   ================================================================
Rem  | List of DLLs that exist for disabled options.                   |
Rem   ================================================================
set Disabled[0\]=oraprtop11.dll.dbl
set Disabled[1\]=oraolapop11.dll.dbl
set Disabled[2\]=oralbac11.dll.dbl
set Disabled[3\]=oradmop11.dll.dbl
set Disabled[4\]=oradv11.dll.dbl
set Disabled[5\]=orarat11.dll.dbl
set Disabled[6\]=clr.dbl

Rem   ================================================================
Rem  | Clear any existing logfile.                                    |
Rem   ================================================================
Rem 
del %MYLOG% > nul 2>&1

call :log %0 - v%VERSION% : Logging to %MYLOG%
call :log Executing: %0 %*

:check_oracle_home
if "%ORA_HOME%" EQU "" (
    set ORA_HOME=%ORACLE_HOME%
)

if "%ORACLE_HOME%" EQU "" (
	call :log ORACLE_HOME is not defined.
	set ERRORS=1
)

if not exist %ORA_HOME% (
    call :log ORACLE_HOME "%ORA_HOME%" - not found.
    set ERRORS=1
)

:check_errors

if %ERRORS% EQU 1 (
	call :log Cannot continue - too many errors.
	goto :eof
)

Rem *******************************************************************
Rem *******************************************************************
:JDI

call :log Checking ORACLE_HOME = "%ORA_HOME%".

for /L %%f in (%FirstEntry%, 1, %LastEntry%) do (

    Rem Is this option enabled?
    if exist %ORA_HOME%\\bin\\!Enabled[%%f\]! (
        call :log !Option[%%f\]! - is currently enabled.
    )
    
    Rem Also check if it is disabled too. This needs investigating.
    if exist %ORA_HOME%\\bin\\!Disabled[%%f\]! (
        call :log !Option[%%f\]! - is currently disabled.
    )
)
Rem *******************************************************************
Rem *******************************************************************

Rem   ================================================================
Rem  | And finally, turn off the "doing it correctly" setting.        |
Rem  | And skip over the sub-routines.                                |
Rem   ================================================================
Rem 
call :log %0 - complete.
endlocal
exit /b

Rem   ================================================================
Rem  |                                                          LOG() |
Rem   ================================================================
Rem  | Set up a logging procedure to log output to the %MYLOG% file.  |
Rem  | Each line is yyyy/mm/dd hh:mi:ss:                       |
Rem   ================================================================
:log

echo %*
echo %date:~6,4%/%date:~3,2%/%date:~0,2% %time:~0,8%: %* >> %MYLOG%
goto :eof
```

A test run gives the following, on screen:

```cmd
C:\\Users\\ndunbar\\Desktop>CheckOracleOptions.cmd
CheckOracleOptions.cmd - v1.00 : Logging to .\\CheckOracleOptions.cmd.log
Executing: CheckOracleOptions.cmd
Checking ORACLE_HOME = "C:\\OracleDatabase\\product\\11.2.0\\dbhome_1".
Partitioning - is currently disabled.
OLAP - is currently disabled.
Label Security - is currently disabled.
Data Mining - is currently disabled.
Database Vault option - is currently disabled.
Real Application Testing - is currently enabled.
Database Extensions for .NET - is currently disabled.
CheckOracleOptions.cmd - complete.
```

And the logfile looks like this:

```
2016/08/25 15:29:58: CheckOracleOptions.cmd - v1.00 : Logging to .\\CheckOracleOptions.cmd.log 
2016/08/25 15:29:58: Executing: CheckOracleOptions.cmd 
2016/08/25 15:29:58: Checking ORACLE_HOME = "C:\\OracleDatabase\\product\\11.2.0\\dbhome_1". 
2016/08/25 15:29:58: Partitioning - is currently disabled. 
2016/08/25 15:29:58: OLAP - is currently disabled. 
2016/08/25 15:29:58: Label Security - is currently disabled. 
2016/08/25 15:29:58: Data Mining - is currently disabled. 
2016/08/25 15:29:58: Database Vault option - is currently disabled. 
2016/08/25 15:29:58: Real Application Testing - is currently enabled. 
2016/08/25 15:29:58: Database Extensions for .NET - is currently disabled. 
2016/08/25 15:29:58: CheckOracleOptions.cmd - complete. 
```

### Oracle 12c Changes

If you are running Oracle 12c on Windows, then the code above needs a couple of minor changes to cope. Oracle have, as usual, changed the names of the various DLLs that need to be checked.

Change the appropriate parts of the above code, to the following:

```cmd
Rem   ================================================================
Rem  | List of DLLs that exist for enabled options.                   |
Rem   ================================================================
set Enabled[0\]=oraprtop12.dll
set Enabled[1\]=oraolapop12.dll
set Enabled[2\]=oralbac12.dll
set Enabled[3\]=oradmop12.dll
set Enabled[4\]=oradv12.dll
set Enabled[5\]=orarat12.dll
set Enabled[6\]=clr

Rem   ================================================================
Rem  | List of DLLs that exist for disabled options.                   |
Rem   ================================================================
set Disabled[0\]=oraprtop12.dll.dbl
set Disabled[1\]=oraolapop12.dll.dbl
set Disabled[2\]=oralbac12.dll.dbl
set Disabled[3\]=oradmop12.dll.dbl
set Disabled[4\]=oradv12.dll.dbl
set Disabled[5\]=orarat12.dll.dbl
set Disabled[6\]=clr.dbl
```

That's all.
