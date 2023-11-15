---
title: "ORA$AT_SA_SPC_SY Jobs failing?"
date: "2013-01-28"
categories: 
  - "oracle"
---

Oracle has raised an alert in the alert.log and created a trace file as well, for a failed `DBMS_SCHEDULER` job with a strange name which doesn't appear in `DBA_SCHEDULER_JOBS` or `DBA_SCHEDULER_PROGRAMS` - what's going on?

An extract from the alert log and/or the trace file mentioned in the alert log shows something like:

```text
*** SERVICE NAME:(SYS.USERS) ...
*** MODULE NAME:(DBMS_SCHEDULER) ...
*** ACTION NAME:(ORA$AT_SA_SPC_SY_nnn) ...
```

Where 'nnn' in the action name is a number.

No matter how hard you scan the `DBA_SCHEDULER_%` views, you will not find anything with this name. What is actually failing?

Oracle 11.1.0.6 onwards stopped listing these internal jobs in `DBA_SCHEDULER_JOBS`, as they did in 10g, and instead lists them in `DBA_AUTOTASK_%` views. However, not by actual name, so don't go looking for a TASK_NAME that matches the above action name. You will fail.

There are three different autotask types:

- Space advisor
- Optimiser stats collection
- SQl tuning advisor

The tasks that run for these autotask 'clients' are named as follows:

- **ORA$AT_SA_SPC_SY_nnn** for Space advisor tasks
- **ORA$AT_OS_OPT_SY_nnn** for Optimiser stats collection tasks
- **ORA$AT_SQ_SQL_SW_nnn** for Space advisor tasks

See MOS notes 756734.1, 755838.1, 466920.1 and Bug 12343947 for details. The first of these has the most relevant and useful information.

> **UPDATE**: My original failing autotask has been diagnosed by Oracle Support as [bug 13840704](https://support.oracle.com/epmos/faces/ui/km/BugDisplay.jspx?id=13840704 "https://support.oracle.com/epmos/faces/ui/km/BugDisplay.jspx?id=13840704") for which a [patch exists here](https://updates.oracle.com/Orion/PatchDetails/process_form?patch_num=13840704 "https://updates.oracle.com/Orion/PatchDetails/process_form?patch_num=13840704") for 11.2.0.2 and 11.2.0.3.
> 
> Oracle document id 13840704.8 has details, but it involves LOBs based on a user defined type. In this case, Spatial data in an `MDSYS.SDO_GEOMETRY` column.

The view `DBA_AUTOTASK_CLIENT` won't show you anything about a specific task, with the above names, but will show you details of what the overall 'client' is, There are three:

```sql
select client_name, status
from dba_autotask_client;

CLIENT_NAME                     STATUS
------------------------------- --------
auto optimizer stats collection ENABLED
auto space advisor              ENABLED
sql tuning advisor              DISABLED
```

I can see from the task name in the alert log and trace file, that my failing task is a space advisor one, so, by looking into the `DBA_AUTOTASK_JOB_HISTORY` view, I can see what's been happening:

```sql
select distinct client_name, window_name, job_status, job_info
from dba_autotask_job_history
where job_status <> 'SUCCEEDED'
order by 1,2;

CLIENT_NAME        WINDOW_NAME     JOB_STATUS JOB_INFO
------------------ --------------- ---------- -------------------------------------------
auto space advisor SATURDAY WINDOW FAILED     ORA-6502: PL/SQL: numeric or value error...
auto space advisor SUNDAY WINDOW   FAILED     ORA-6502: PL/SQL: numeric or value error...
```

So, in my own example, the auto space advisor appears to have failed on Saturday and Sunday. Given that this is an internal task, and nothing I can do will let me know about the invalid number problem, I need to log an SR with Oracle on the matter. However, as I don't want my fellow DBAs to be paged in the wee small hours for a known problem, I have disabled the space advisor task as follows:

```sql
BEGIN
  dbms_auto_task_admin.disable(
    client_name => 'auto space advisor',
    operation   => NULL,
    window_name => NULL);
END;
/

PL/SQL procedure successfully completed
```

Checking `DBA_AUTOTASK_CLIENT` again, shows that it is indeed disabled:

```sql
select client_name, status
from dba_autotask_client
where client_name = 'auto space advisor';

CLIENT_NAME                     STATUS
------------------------------- --------
auto space advisor              DISABLED
```

Enabling it again after Oracle Support have helped resolve the problem is as simple as calling `dbms_auto_task_admin.enable` with exactly the same parameters as for the disable call:

```sql
BEGIN
  dbms_auto_task_admin.enable(
    client_name => 'auto space advisor',
    operation   => NULL,
    window_name => NULL);
END;
/

PL/SQL procedure successfully completed
```

When enabling and/or disabling auto tasks, you _must_ use the `CLIENT_NAME` as found in `DBA_AUTOTASK_CLIENT` view.

The full list of DBA_AUTOTASK_% views is:

- `DBA_AUTOTASK_CLIENT`
- `DBA_AUTOTASK_CLIENT_HISTORY`
- `DBA_AUTOTASK_CLIENT_JOB`
- `DBA_AUTOTASK_JOB_HISTORY`
- `DBA_AUTOTASK_OPERATION`
- `DBA_AUTOTASK_SCHEDULE`
- `DBA_AUTOTASK_TASK`
- `DBA_AUTOTASK_WINDOW_CLIENTS`
- `DBA_AUTOTASK_WINDOW_HISTORY`

Hope this helps!
