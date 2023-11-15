---
title: "Shell Variable Indirection in a Database Build Script"
date: "2014-11-11"
categories: 
  - "linux"
  - "oracle"
---

Ever wanted to set a variable to the name of another variable, and from there, somehow get the value of the other variable? I did, recently, and this is what I had to do.

I work with numerous databases but of all the ones I have, there are only 18 different types and these cover all possible (at present) systems in production or development. The first 3 characters of `$ORACLE_SID` define the system name and we use a script that duplicates any of 18 template databases to create the desired new one. The script has to run on both the Bash and Korn shells, and must validate that the new database is going to be built from the correct database template - to catch DBA finger troubles! :)

The template databases exist as RMAN backups and have been created with the correct options, default tablespaces, users and all the desired options for the system, so after the build is complete, and the post-build validation script has been executed, the new database can be handed over to the users without any further changes. Passwords are set up randomly after the build has completed, by the build script, so anyone who knows the template database passwords won't know them on a new build.

The script is passed a template database name on the command line and this needed to be validated to ensure that ORACLE_SID, the new database name, was permitted to be built with that particular template.

```bash
$ myscript.sh -t XXX ...
```

The script already validates `XXX` as one of the 18 allowed values, but a recent change means that now, the script needs to carry out 1 of 18 different validations depending on the `XXX` parameter passed at run time.

How difficult could it be? As it turned out, not very - once you know about variable indirection in bash, and the following code also works in the Korn shell which I also needed it to work on.

### The Obligatory Hello World Example

The following example can be typed in at the Bash or Korn shell prompt.

```bash
ABC="Hello World"
X="ABC"
eval Y=\$"${X}"
echo "${Y}"

Hello World
```

Variable $ABC holds the _value_ I am after, $X holds the _name_ of the variable that holds the _value_ I want. The `eval` function sets variable $Y to the _value_ held in the variable whose _name_ is held in $X. So $Y is set to "Hello World". Simple!

### The Database Creation Script

In my database creation script, all I had to do was set up a one variables for each of the different template databases allowed. The name of the variable had to match the name of the template database that would be passed in, in upper case, by the `-t` parameter.

```bash
duplicate_database -t T_7 -o $ORACLE_SID ...
```

In the validation function, all I had to do was get the correct list of valid database prefixes into a separate variable using indirection, and from there it was a simple case of `grep`ing to see if the desired system prefix was present in the allowed list.

The code looked remarkably similar to the following and, as ever, systems and database names are not based on reality - to protect the innocent!

```bash
...
#----------------------------------------------------------------------------
# The following list of variables holds the permitted 3 character prefix for 
# a database name created with the appropriate template.
#
# For example, If the template is T_1, only databases named SYSxxxxx and
# DBAxxxxx are valid.
#----------------------------------------------------------------------------
T_1="|SYS|DBA|"
T_2="|PAY|HMN|SOP|"
...
T_18="|PRE|RTL|XXX|DAT|"

#----------------------------------------------------------------------------
# A function to validate the database name passed in against a database type.
#----------------------------------------------------------------------------
# $1 is the Database Template passed on the command line with "-t T_1" etc.
# $2 is the database name to be built. (aka ORACLE_SID)
# 
# Note, this uses indirection and so the database type MUST match the name of a
# validation variable set up previously.
#
# Return Code: 
# $? = 0 if first three characters of ORACLE_SID are valid for the template.
# $? = 1 if not.
#----------------------------------------------------------------------------
validate_db_template()
{
   #----------------------------------------------------
   # Make sure both parameters come to us in upper case.
   #----------------------------------------------------
   DB_TEMPLATE=`echo $1 | tr '[:lower:]' '[:upper:]'`
   DB_PREFIX=`echo $2 | tr '[:lower:]' '[:upper:]'`

   #----------------------------------------------------
   # DB_TEMPLATE now holds the upper case T_1 .. T_18 
   # name of a template database that we will use to 
   # build a new database as per ORACLE_SID/$2/DB_PREFIX.
   # We now need to get a list of the valid database 
   # names for this template.
   #----------------------------------------------------
   eval VALID_LIST=\$"${DB_TEMPLATE}"

   #----------------------------------------------------
   # We only need the first 3 characters of the DB name.
   #----------------------------------------------------
   DB_PREFIX=`echo "${DB_PREFIX}"|cut -c 1-3`

   #----------------------------------------------------
   # Then check if it is in VALID_LIST.
   # $? = 0 if found, 1 if not.
   #----------------------------------------------------
   echo "${VALID_LIST}" | grep -qi "${DB_PREFIX}"
   return $?
}
```

To validate that a passed in template database permits the database named as per ORACLE_SID to be created, it was a simple matter to call `validate_db_template` passing the desired parameters, and check the value in `$?` after the function had returned:

```bash
...
validate_db_template "${TEMPLATE}" "${ORACLE_SID}"

if [ "${?}" != "0" ]
then
   echo "*** ERROR: validate_db_template failed"
   echo "*** ${ORACLE_SID} cannot be built with a database template of \"${TEMPLATE}\"."
   exit "${ERROR_DBNAME_VALIDATION_FAILED}"
fi
...
```

Using indirection in this manner saved me the horror of typing in a huge `case` statement where I would set the VALID_LIST according to what the current template name was. In addition, future amendments, perhaps for Oracle 12c, will simply require a new template variable and allowed systems to be created, no code need be changed.

You might not need to have a database duplication script like I have, but I'm sure that the ability to get the value of a variable whose name is unknown until run time, might prove useful.

Have fun.
