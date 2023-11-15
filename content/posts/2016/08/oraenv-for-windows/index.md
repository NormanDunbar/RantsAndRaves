---
title: "Oraenv for Windows"
date: "2016-08-26"
categories: 
  - "oracle"
  - "windows"
---

Having recently had to learn a whole new way of working when I took on a contract migrating a database to the Windows "cloud", I realised that there's no equivalent to the useful Unix `oraenv` utility. I had to write my own. Give me a `bash` shell any day!

The utility is `oraenv.cmd` and executes like this:

```
set ORATAB=c:\\users\\ndunbar\\oratab
...
oraenv
```

> **Update 30/08/2016:** You can now pass the desired SID on the command line and avoid all that prompting stuff! Like this:
> 
>```
> oraenv AZDBA01
> ...
>```

Obviously, the `ORATAB` environment variable can be set in Control Panel, or in the shell session previously etc. As long as it is set somewhere.

The `%ORATAB%` file needs to look like the following:

```
ORACLE_SID | ORACLE_HOME | Optional comment text.
```

The default Unix separator of a colon, ':', cannot be used here as the ORACLE_HOME field will no doubt have a colon in its name, given that Windows uses it as part of the drive specification. To get around that foible, I use a pipe character - '|'.

My own oratab file looks like this:

```
AZDBA01|C:\\OracleDatabase\\product\\11.2.0\\dbhome_1|# Staging database
AZDBA02|C:\\OracleDatabase\\product\\11.2.0\\dbhome_1|# Test clone
AZDBA91|C:\\OracleDatabase\\product\\11.2.0\\dbhome_1|# Standby for AZDBA01
AZDEV08|C:\\OracleDatabase\\product\\11.2.0\\dbhome_1|# Development
AZDEV12|C:\\OracleDatabase\\product\\11.2.0\\dbhome_1|# Development
```

Comments are not mandatory, but if present, there _must_ be a pipe character - '|' - after the end of the Oracle Home or the comment becomes part of the `%ORACLE_HOME%` environment variable if not. Ask me how I know this!

In use, an example of the utility's output would be similar to the following:

```
C:\\Users\\ndunbar>oraenv
Your session's current Oracle SID is 'azdba02'.

Please enter a new Oracle SID from the following list:
AZDBA01
AZDBA02
AZDBA91
AZDEV08
AZDEV12

Press ENTER/RETURN to use the current ORACLE_SID.
New SID \[azdba02\]: azdba91
ORACLE_HOME\\bin is already on PATH.
ORACLE_SID has been set to 'azdba91'.
ORACLE_HOME has been set to 'C:\\OracleDatabase\\product\\11.2.0\\dbhome_1'.
NLS_LANG has been set to 'AMERICAN_AMERICA.WE8ISO8859P1'.
NLS_DATE_FORMAT has been set to 'yyyy/mm/dd hh24:mi:ss'.
```

As noted, the current `%ORACLE_SID%` is the default and pressing `RETURN` accepts that and leaves things unchanged.

If the desired `%ORACLE_HOME%` is on the `%PATH%` already, it is not added again. And this identifies a **problem**.

If there are numerous different Oracle Homes in use on the server, then the new one will be added to `%PATH%` but the old one(s) will not, at present, be removed. I need to find a decent stream editor - like `sed` - for Windows to allow me the opportunity to do that. However, in my current installations, we have a single Oracle Home for all our databases on the servers, so that problem isn't affecting me at present. Famous last words?

In the event of a problem, the following error codes are returned:

- 0 = All ok.
- 1 = ORATAB environment variable not set.
- 2 = %ORATAB% not pointing to an (accessible) file.
- 3 = Requested Oracle SID not found in %ORATAB% file.

Anyway, here's the code. Copy this and paste into your own `oraenv.cmd`, somewhere on your `%path%`, and you are all set to go. Don't forget to set `ORATAB` first though.

Enjoy.


```cmd
@echo off

REM =================================================================
REM Windows version, sort of, of the Unix oraenv script which
REM will set the desired Oracle Environment.
REM 
REM Requires the %ORATAB% environment variable pointing at a suitable
REM text file, which has the lines set up in the following format:
REM
REM SID | ORACLE_HOME
REM
REM We can't use the Unix default separator of a colon (:) as that 
REM is used already for the drive specifiers.
REM
REM EXIT CODES:
REM
REM 0 = All ok, environment set as requested.
REM 1 = Oops! ORATAB env var not set.
REM 2 = Oops! %ORATAB% not pointing to an accessible file.
REM 3 = Oops! Requested ORACLE_SID not found in %ORATAB% file.
REM
REM =================================================================
REM Norman Dunbar (norman@dunbar-it.co.uk)
REM June 2016.
REM
REM 30/08/2016 - Added ability to pass SID on command line.
REM =================================================================

REM Check if %ORATAB% is already set. Bail out if not.
REM This could be set in the System applet for Control Panel,
REM or, set in the shell prior to calling this code.
if "%ORATAB%"=="" (
    echo ORATAB not set. Cannot continue.
    exit /b 1
)

REM ORATAB needs to point at a file.
if NOT exist %ORATAB% (
    echo Cannot find the file '%ORATAB%'. Cannot continue.
    echo Check the value in ORATAB is correctly set, or, that
    echo the file exists.
    exit /b 2
)

REM Did we have a SID passed as a parameter?
set ORA_SID=%1

REM We don't do the next bit if we have a SID already.
if "%ORA_SID%"=="" (

    REM Display the Current ORACLE_SID.
    echo Your session's current Oracle SID is '%oracle_sid%'.
    SET ORA_SID=%oracle_sid%
    echo.

    REM List the available SIDs from the oratab file.
    echo Please enter a new Oracle SID from the following list:
    for /f "tokens=1 delims=|" %%a in (%ORATAB%) do (
        echo %%a
    )
    echo.

    REM Default to the current SID if the user just presses ENTER.
    echo Press ENTER/RETURN to use the current ORACLE_SID.
    SET /P ORA_SID="New SID \[%ORA_SID%?\]: "
)

REM Check the %ORATAB% file to see if this ORACLE_SID is listed
REM If it's not then exit to the command prompt with error 3.
FIND /I "%ORA_SID%|" %ORATAB% > nul
IF NOT %ERRORLEVEL%==0 (
    echo Oracle SID not found
    exit /b 3
)

REM Set the ORACLE_SID.
set ORACLE_SID=%ORA_SID%

REM Now get the Oracle Home from the oratab file.
FOR /F "tokens=2 delims=|" %%a IN ('FIND /I "%ORA_SID%|" %ORATAB%') DO SET ORACLE_HOME=%%a

REM Next thing to do is set the Path. Ok, possible problem area!
REM In my own environment, everything has the same ORACLE_HOME so
REM there's no need to worry about removing any other ORACLE_HOME
REM from the PATH before adding this one. I need to think about this.
echo %PATH% | find /i "%ORACLE_HOME%\\bin" > nul
if NOT %ERRORLEVEL%==0 (
    SET PATH=%ORACLE_HOME%\\bin;%PATH%
) else (
    echo ORACLE_HOME\\bin is already on PATH. 
)    

REM And as a nice little touch we will Print out some details for the user.
ECHO ORACLE_SID has been set to '%ORACLE_SID%'.
ECHO ORACLE_HOME has been set to '%ORACLE_HOME%'.

REM Uncomment this if you want your NLS_LANG and NLS_DATE_FORMATs setting.
set NLS_LANG=AMERICAN_AMERICA.WE8ISO8859P1
echo NLS_LANG has been set to '%NLS_LANG%'.

set NLS_DATE_FORMAT=yyyy/mm/dd hh24:mi:ss
echo NLS_DATE_FORMAT has been set to '%NLS_DATE_FORMAT%'.

exit /b 0
@echo on
```
