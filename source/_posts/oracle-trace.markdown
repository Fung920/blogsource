---
layout: post
title: "Oracle追踪SQL的方法"
date: 2014-07-24 15:15:11
comments: false
categories: oracle
tags: tuning
keywords: trace,tkprof,10046,10053
description: how to trace sql or others in oracle
---
Oracle提供了几种方法可以使得DBA针对某些session或者语句进行trace跟踪。
<!--more-->
从9i开始，官方文档开始记载了如何激活SQL追踪的方法，初始化参数SQL_TRACE,dbms_session.set_sql_trace和dbms_system.set_sql_trace_in_session。然而，这三种方法都只能追踪到10046事件的1级别，10046级别如下图所示（不完全，11g有新增10046级别）：

<center>Table 1-1 10046 event level</center>

|level|description|
|----|----|
|0|禁止trace|
|1|激活trace，提供SQL语句，响应时间，服务时间，处理行数，逻辑读，物理读写，执行计划及额外的一些信息|
|4|包括1级别，同时提供绑定变量信息|
|8|包括1级别，同时提供等待事件信息：等待事件名字、持续时间等|
|12|同时启动4和8|

由于官方文档提及的三种追踪方法都无法更详细的信息，因此，有如下几种方式可提供对SQL或者其他事件的追踪。
### 1. event事件
对SQL的追踪主要有10046及10053事件，这两个事件都是不公开的，解析分别如下(10053仅有两个级别，一般会使用level 1，属于最高级别)：
```
[oracle@linora:/home/oracle]$ oerr ora 10046
10046, 00000, "enable SQL statement timing"
// *Cause:
// *Action:
[oracle@linora:/home/oracle]$ oerr ora 10053
10053, 00000, "CBO Enable optimizer trace"
// *Cause:
// *Action:
```
10046显示SQL执行计划如何运行， 10053显示优化器为何选择这个执行计划。通过设置Event对SQL进行追踪，可以有两种方法，一种是alter session/alter system方法，或者使用oradebug工具。
#### 1.1 alter session/alter system
```
--开启session级别事件追踪
alter session set events '10046 trace name context forever,level 12';
alter session set events '10053 trace name context forever,level 1';
--关闭事件
ALTER SESSION SET EVENTS '10053 trace name context off';
```
查找trace文件：
```
#for 11g
SELECT s.sql_trace, s.sql_trace_waits, s.sql_trace_binds,
traceid, tracefile
FROM v$session s JOIN v$process p ON (p.addr = s.paddr)
WHERE audsid = USERENV ('SESSIONID');
#for 10g
SELECT   s.sid,
         s.serial#,
            pa.VALUE
         || '/'
         || LOWER (SYS_CONTEXT ('userenv', 'instance_name'))
         || '_ora_'
         || p.spid
         || '.trc'
            AS trace_file
  FROM   v$session s, v$process p, v$parameter pa
 WHERE       pa.name = 'user_dump_dest'
         AND s.paddr = p.addr
         AND s.audsid = SYS_CONTEXT ('USERENV', 'SESSIONID');
```
#### 1.2 oradebug
#####oradebug追踪本session
oradebug的用法可用oradebug help查看，关于oradebug其他功能，本文暂不予讨论。
```
SYS@linora> oradebug setmypid
Statement processed.
SYS@linora> oradebug event 10053 trace name context forever,level 1; 
Statement processed.
SYS@linora> oradebug event 10053 trace name context off
Statement processed.
SYS@linora> oradebug tracefile_name
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_2293.trc
SYS@linora> oradebug CLOSE_TRACE
```
#####oradebug追踪其他session
如果是OS系统进程ID，可使用oradebug setospid spid 。如果是Oracle ID可使用oradebug setorapid pid 进行追踪。SPID(system process id),表示該server process在OS层面的Porcess ID 即操作系统进程ID,PID(Oracle process id) 可以理解为Oracle自己用的，Oracle进程ID,SID(SESSION)标识，常用于连接其它列。首先通过v$session和v$process查找相关的pid或者spid：
```
SYS@linora> SELECT  a.sid,a.serial#,b.spid,a.username,a.machine,a.OSUSER,a.PROGRAM
FROM   v$session a,v$process b
WHERE   a.TYPE != 'BACKGROUND' and a.paddr=b.addr;

       SID    SERIAL# SPID       USERNAME   MACHINE              OSUSER          PROGRAM
---------- ---------- ---------- ---------- -------------------- --------------- ------------------------------------------------
        26         37 2552       FUNG       linora               oracle          sqlplus@linora (TNS V1-V3)
       147        195 3316       SYS        linora               oracle          sqlplus@linora (TNS V1-V3)
       144        179 3318                  linora               oracle          oracle@linora (J000)
        20        101 3192       FUNG       WORKGROUP\PC0920     Administrator   sqlplus.exe
        22         23 3320                  linora               oracle          oracle@linora (J001)
```
比如，要对sid=20,serial#=101,spid=3192的进程进行跟踪
```
SYS@linora> oradebug setospid 3192
Oracle pid: 30, Unix process pid: 3192, image: oracle@linora
#或者oradebug setorapid 20
SYS@linora> oradebug event 10046 trace name context forever,level 12; 
Statement processed.
SYS@linora> oradebug event 10046 trace name context off
Statement processed.
SYS@linora> oradebug tracefile_name
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_3192.trc
SYS@linora> oradebug CLOSE_TRACE
Statement processed.
```
### 2. dbms_system.set_ev
这个procedure是为10g以前提供的，在我的环境下怎么也找不到生成的trace file，以下语句使用dbms包开启SQL追踪（默认sys用户执行）：
```
SYS@linora> EXEC DBMS_SYSTEM.set_ev(si=>20, se=>101, ev=>10046, le=>12, nm=>' ');
PL/SQL procedure successfully completed.
```
关闭只需要将le改为0即可：
```
SYS@linora> EXEC DBMS_SYSTEM.set_ev(si=>20, se=>101, ev=>10046, le=>0, nm=>' ');
PL/SQL procedure successfully completed.
```
这个包对应的解析如下：
```
si binary_integer, -- SID
se binary_integer, -- Serial#
ev binary_integer, -- Event code or number to set.
le binary_integer, -- Usually level to trace
nm varchar2 -- sets the Event Name. null = "context forever".
```
### 3. dbms_monitor
10g提供了一个新的包dbms_monitor可以用来启停SQL的追踪。使用这个包，不只是最终有一个官方的方式来全面利用SQL追踪，而且能够根据会话属性—客户端标记、服务名、模块名及操作名，开启或者关闭SQL追踪。默认情况下，只有DBA角色才运行执行这个包提供的过程。  
会话级别dbms_monitor分别提供了session_trace_enable和session_trace_disable两个存储过程。  
在10gR2中，当用session_trace_enbale开启追踪的时候，v$session视图的列sql_trace, sql_trace_waits, sql_trace_binds也会被设定跟上述存储过程相应的值。
```
SYS@linora> exec dbms_monitor.session_trace_enable(session_id => 20, serial_num => 101, waits => TRUE, binds => TRUE);
PL/SQL procedure successfully completed.
SYS@linora> SELECT sql_trace, sql_trace_waits, sql_trace_binds
  2  FROM v$session
  3  WHERE sid = 20;
SQL_TRAC SQL_T SQL_T
-------- ----- -----
ENABLED  TRUE  TRUE
```
停止trace：
```
SYS@linora> exec dbms_monitor.session_trace_disable(session_id => 20, serial_num => 101);
PL/SQL procedure successfully completed.
SYS@linora> SELECT sql_trace, sql_trace_waits, sql_trace_binds
  2  FROM v$session
  3  WHERE sid = 20;
SQL_TRAC SQL_T SQL_T
-------- ----- -----
DISABLED FALSE FALSE
```
### 4. tkprof
tkprof是格式化trace文件的一个工具，裸trace文件，读起来会有点困难，Oracle提供tkprof工具进行更为友好的阅读trace文件.此工具用法比较简单，在无其他参数的情况下，直接敲<code>tkprof</code>会得到简略的使用说明。注意，tkprof仅仅是作为对SQL跟踪的一种格式化工具，像10053或者其他错误信息导致的trace文件，tkprof是读取不了其中的信息的。
```
SCOTT@linora> alter session set events '10046 trace name context forever,level 12';
Session altered.
SCOTT@linora> select ename,job,sal,dname
  2  from emp,dept where dept.deptno = emp.deptno and not exists
  3  (select * from salgrade where emp.sal between losal and hisal);
no rows selected
SCOTT@linora> ALTER SESSION SET EVENTS '10046 trace name context off';
Session altered.
[oracle@linora:/home/oracle]$ tkprof /u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_3899.trc \
10046.txt waits=yes aggregate=yes sys=no
```
有几个参数有必要说明一下：
<li><code>explain</code></li>
explain=username/password，如果没有使用explain，在trace文件中看到的会是SQL的实际执行路径，如果使用了explain，则在trace不但输入了实际的执行路径，还会生成该SQL的执行计划。  
<li><code>sys</code></li>
sys=yes/no，默认为yes，表示在trace文件中输出sys用户的操作，设置sys=no后，trace文件更具有可读性。  
<li><code>aggregate</code></li>
aggregate=yes/no，默认设置为yes，即将所有相同的SQL在输入文件中合并，如果设置为no，则分别列出每个sql的信息。
<li><code>waits</code></li>
waits=yes/no，waits=yes显示等待事件及等待时间信息等。
<li><code>sort</code></li>
sort=option有大量的排序条件，一般对于查询的调优 常用的组合是 SYS=NO  fchela， fchela即按照fetch阶段的elapsed time按照从大到小排列，fchela也是默认的方式。
```
Rows (1st) Rows (avg) Rows (max)  Row Source Operation
---------- ---------- ----------  ---------------------------------------------------
         0          0          0  MERGE JOIN ANTI (cr=15 pr=0 pw=0 time=2769 us cost=11 size=840 card=14)
        14         14         14   SORT JOIN (cr=8 pr=0 pw=0 time=1118 us cost=7 size=476 card=14)
        14         14         14    MERGE JOIN  (cr=8 pr=0 pw=0 time=839 us cost=6 size=476 card=14)
         4          4          4     TABLE ACCESS BY INDEX ROWID DEPT (cr=2 pr=0 pw=0 time=323 us cost=2 size=52 card=4)
         4          4          4      INDEX FULL SCAN PK_DEPT (cr=1 pr=0 pw=0 time=86 us cost=1 size=0 card=4)(object id 84557)
        14         14         14     SORT JOIN (cr=6 pr=0 pw=0 time=430 us cost=4 size=294 card=14)
        14         14         14      TABLE ACCESS FULL EMP (cr=6 pr=0 pw=0 time=175 us cost=3 size=294 card=14)
        14         14         14   FILTER  (cr=7 pr=0 pw=0 time=1420 us)
        14         14         14    SORT JOIN (cr=7 pr=0 pw=0 time=699 us cost=4 size=130 card=5)
         5          5          5     TABLE ACCESS FULL SALGRADE (cr=7 pr=0 pw=0 time=114 us cost=3 size=130 card=5)


Elapsed times include waiting on following events:
  Event waited on                             Times   Max. Wait  Total Waited
  ----------------------------------------   Waited  ----------  ------------
  SQL*Net message to client                       1        0.00          0.00
```
以上为实际的执行路径，以下为带执行计划的实际执行路径：
```
$ tkprof linora_ora_2392.trc explain_10046.txt sys=no aggregate=yes waits=yes explain=scott/oracle
Rows (1st) Rows (avg) Rows (max)  Row Source Operation
---------- ---------- ----------  ---------------------------------------------------
         0          0          0  MERGE JOIN ANTI (cr=15 pr=0 pw=0 time=2769 us cost=11 size=840 card=14)
        14         14         14   SORT JOIN (cr=8 pr=0 pw=0 time=1118 us cost=7 size=476 card=14)
        14         14         14    MERGE JOIN  (cr=8 pr=0 pw=0 time=839 us cost=6 size=476 card=14)
         4          4          4     TABLE ACCESS BY INDEX ROWID DEPT (cr=2 pr=0 pw=0 time=323 us cost=2 size=52 card=4)
         4          4          4      INDEX FULL SCAN PK_DEPT (cr=1 pr=0 pw=0 time=86 us cost=1 size=0 card=4)(object id 84557)
        14         14         14     SORT JOIN (cr=6 pr=0 pw=0 time=430 us cost=4 size=294 card=14)
        14         14         14      TABLE ACCESS FULL EMP (cr=6 pr=0 pw=0 time=175 us cost=3 size=294 card=14)
        14         14         14   FILTER  (cr=7 pr=0 pw=0 time=1420 us)
        14         14         14    SORT JOIN (cr=7 pr=0 pw=0 time=699 us cost=4 size=130 card=5)
         5          5          5     TABLE ACCESS FULL SALGRADE (cr=7 pr=0 pw=0 time=114 us cost=3 size=130 card=5)
Rows     Execution Plan
-------  ---------------------------------------------------
      0  SELECT STATEMENT   MODE: ALL_ROWS
      0   NESTED LOOPS
     14    NESTED LOOPS
     14     MERGE JOIN (ANTI)
      4      SORT (JOIN)
      4       TABLE ACCESS   MODE: ANALYZED (FULL) OF 'EMP' (TABLE)
     14      FILTER
     14       SORT (JOIN)
     14        TABLE ACCESS   MODE: ANALYZED (FULL) OF 'SALGRADE' (TABLE)
     14     INDEX   MODE: ANALYZED (UNIQUE SCAN) OF 'PK_DEPT' (INDEX(UNIQUE))
      5    TABLE ACCESS   MODE: ANALYZED (BY INDEX ROWID) OF 'DEPT'(TABLE)
Elapsed times include waiting on following events:
  Event waited on                             Times   Max. Wait  Total Waited
  ----------------------------------------   Waited  ----------  ------------
  SQL*Net message to client                       1        0.00          0.00
```
### 5. 10046 trace文件阅读
#####经过tkprof格式化后的文件：
```
TKPROF: Release 11.2.0.4.0 - Development on Thu Jul 24 23:17:09 2014
Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.
Trace file: linora_ora_2348.trc
Sort options: default
********************************************************************************
count    = number of times OCI procedure was executed
cpu      = cpu time in seconds executing 
elapsed  = elapsed time in seconds executing
disk     = number of physical reads of buffers from disk
query    = number of buffers gotten for consistent read
current  = number of buffers gotten in current mode (usually for update)
rows     = number of rows processed by the fetch or execute call
********************************************************************************
SQL ID: 8gt5ty6pfca35 Plan Hash: 1070587533

SELECT /* OPT_DYN_SAMP */ /*+ ALL_ROWS IGNORE_WHERE_CLAUSE 
  NO_PARALLEL(SAMPLESUB) opt_param('parallel_execution_enabled', 'false') 
  NO_PARALLEL_INDEX(SAMPLESUB) NO_SQL_TUNE */ NVL(SUM(C1),:"SYS_B_0"), 
  NVL(SUM(C2),:"SYS_B_1"), COUNT(DISTINCT C3), NVL(SUM(CASE WHEN C3 IS NULL 
  THEN :"SYS_B_2" ELSE :"SYS_B_3" END),:"SYS_B_4"), COUNT(DISTINCT C4), 
  NVL(SUM(CASE WHEN C4 IS NULL THEN :"SYS_B_5" ELSE :"SYS_B_6" END),
  :"SYS_B_7") 
FROM
 (SELECT /*+ NO_PARALLEL("SALGRADE") FULL("SALGRADE") 
  NO_PARALLEL_INDEX("SALGRADE") */ :"SYS_B_8" AS C1, :"SYS_B_9" AS C2, 
  "SALGRADE"."HISAL" AS C3, "SALGRADE"."LOSAL" AS C4 FROM "SCOTT"."SALGRADE" 
  "SALGRADE") SAMPLESUB
```
上述信息包含了tkprof版本信息，trace文件名称，以及在一些图表中各个参数的解析。在上例中，我们看到有<code>/\* OPT_DYN_SAMP \*/</code>信息，表示这个语句是CBO在做动态采样的SQL语句，也表明，我们这个SALGRADE表可能没做分析。Misses in library cache during 表示在解析及执行阶段library cache发生了miss，则说明是硬解析。  
call: 每一个游标的行为被分成三个步骤：  
<code>Parse</code>：表示SQL的分析阶段所耗资源。  
<code>Execute</code>：表示SQL的执行阶段所耗资源。  
<code>Fetch</code>：表示数据提取阶段所耗资源。  
<code>Count</code>：表示当前的操作被执行了多少次。  
<code>Cpu</code>：表示当前操作消耗的CPU事件，单位为秒。  
<code>Elapsed</code>：表示当前操作一共用时多少秒，包括CPU事件跟等待时间。  
<code>Disk</code>：表示当前操作的物理读，即磁盘IO次数。  
<code>Query</code>：表示当前操作的一致性读方式读取的数据块，通常是查询语句使用的方式。  
<code>Current</code>：表示当前操作的逻辑读取的数据块，通常是修改数据使用的方式。  
<code>Rows</code>：表示当前操作处理的数据记录数。  
```
select  ename,job,sal,dname
from emp,dept where dept.deptno = emp.deptno and not exists
(select * from salgrade where emp.sal between losal and hisal)
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.02       0.02          0         22          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        1      0.00       0.00          0         15          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        3      0.02       0.02          0         37          0           0
Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 70  (SCOTT)
Number of plan statistics captured: 1
```
接下来就是上面的我们执行的相关SQL信息，可以从上面的信息看到，优化器模式使用的是ALL_ROWS，且有一次硬解析，用户为SCOTT。此条SQL语句被分析了一次，执行了一次，数据提取了一次，消耗CPU时间为0.02秒，均花在解析阶段，一致性读取了37个块，没有磁盘读。
#####未经tkprof格式化的文件
构建环境，加入一个绑定变量并生成trace
```
FUNG@linora> VARIABLE loc VARCHAR2(30)
FUNG@linora> EXEC :loc:='South San Francisco'
PL/SQL procedure successfully completed.
FUNG@linora> alter session set events '10046 trace name context forever, level 12';
Session altered.
FUNG@linora> SELECT emp.last_name, emp.first_name, j.job_title, d.department_name, l.city,
l.state_province, l.postal_code, l.street_address, emp.email,
emp.phone_number, emp.hire_date, emp.salary, mgr.last_name
FROM hr.employees emp, hr.employees mgr, hr.departments d, hr.locations l, hr.jobs j
WHERE l.city=:loc
AND emp.manager_id=mgr.employee_id
AND emp.department_id=d.department_id
AND d.location_id=l.location_id
AND emp.job_id=j.job_id;
FUNG@linora> select * from v$diag_info;
```
未经tkprof格式化的trace文件
```
PARSING IN CURSOR # 140430523695752 len=430 dep=0 uid=66 oct=3 lid=66 tim=1406340835532883 hv=2605724446 ad='8b986bb8' sqlid='6fdr5n2dp0csy'
SELECT emp.last_name, emp.first_name, j.job_title, d.department_name, l.city,
l.state_province, l.postal_code, l.street_address, emp.email,
emp.phone_number, emp.hire_date, emp.salary, mgr.last_name
FROM hr.employees emp, hr.employees mgr, hr.departments d, hr.locations l, hr.jobs j
WHERE l.city=:loc
AND emp.manager_id=mgr.employee_id
AND emp.department_id=d.department_id
AND d.location_id=l.location_id
AND emp.job_id=j.job_id
END OF STMT
PARSE # 140430523695752:c=3000,e=3186,p=0,cr=0,cu=0,mis=1,r=0,dep=0,og=1,plh=0,tim=1406340835532850
```
CURSOR表示游标号，len表示SQL语句长度，dep表示SQL的递归深度，如果dep=0，表示不是递归SQL，UID为users$表中的user#，通过它可以查询到是谁发起的SQL，OCT=3为sql命令的类型，3表示select操作，11g可通过select command_type,command_name from V$SQLCOMMAND;获得对应关系，c和e都表示时间，一个是cpu消耗时间，一个是Elapsed时间，都以1/100W秒（微秒）为单位。
```
WAIT # 140430523695752: nam='db file sequential read' ela= 9559 file#=8 block#=11515 blocks=1 obj#=84744 tim=1406340835774246
WAIT # 140430523695752: nam='db file sequential read' ela= 503 file#=8 block#=11476 blocks=1 obj#=84737 tim=1406340835774830
WAIT # 140430523695752: nam='db file scattered read' ela= 671 file#=8 block#=11477 blocks=3 obj#=84737 tim=1406340835776772
WAIT # 140430523695752: nam='db file sequential read' ela= 496 file#=8 block#=11458 blocks=1 obj#=84735 tim=1406340835777504
WAIT # 140430523695752: nam='db file scattered read' ela= 560 file#=8 block#=11459 blocks=5 obj#=84735 tim=1406340835778142
WAIT # 140430523695752: nam='db file sequential read' ela= 519 file#=8 block#=11490 blocks=1 obj#=84739 tim=1406340835779259
WAIT # 140430523695752: nam='db file scattered read' ela= 888 file#=8 block#=11491 blocks=5 obj#=84739 tim=1406340835780251
WAIT # 140430523695752: nam='db file sequential read' ela= 12191 file#=8 block#=11666 blocks=1 obj#=84747 tim=1406340835793607
WAIT # 140430523695752: nam='db file scattered read' ela= 725 file#=8 block#=11667 blocks=5 obj#=84747 tim=1406340835794438
FETCH # 140430523695752:c=7000,e=60243,p=27,cr=19,cu=0,mis=0,r=1,dep=0,og=1,plh=2361458858,tim=1406340835794675
WAIT # 140430523695752: nam='SQL*Net message from client' ela= 458 driver id=1650815232 #bytes=1 p3=0 obj#=84747 tim=1406340835795214
WAIT # 140430523695752: nam='SQL*Net message to client' ela= 5 driver id=1650815232 #bytes=1 p3=0 obj#=84747 tim=1406340835795283
FETCH # 140430523695752:c=0,e=69,p=0,cr=1,cu=0,mis=0,r=15,dep=0,og=1,plh=2361458858,tim=1406340835795333
WAIT # 140430523695752: nam='SQL*Net message from client' ela= 2045 driver id=1650815232 #bytes=1 p3=0 obj#=84747 tim=1406340835797425
WAIT # 140430523695752: nam='SQL*Net message to client' ela= 7 driver id=1650815232 #bytes=1 p3=0 obj#=84747 tim=1406340835797553
FETCH # 140430523695752:c=0,e=141,p=0,cr=1,cu=0,mis=0,r=15,dep=0,og=1,plh=2361458858,tim=1406340835797621
WAIT # 140430523695752: nam='SQL*Net message from client' ela= 9520 driver id=1650815232 #bytes=1 p3=0 obj#=84747 tim=1406340835807188
WAIT # 140430523695752: nam='SQL*Net message to client' ela= 5 driver id=1650815232 #bytes=1 p3=0 obj#=84747 tim=1406340835807262
FETCH # 140430523695752:c=0,e=149,p=0,cr=1,cu=0,mis=0,r=14,dep=0,og=1,plh=2361458858,tim=1406340835807391
STAT # 140430523695752 id=1 cnt=45 pid=0 pos=1 obj=0 op='HASH JOIN  (cr=22 pr=27 pw=0 time=60289 us cost=10 size=2580 card=15)'
STAT # 140430523695752 id=2 cnt=45 pid=1 pos=1 obj=0 op='HASH JOIN  (cr=13 pr=15 pw=0 time=43883 us cost=8 size=2400 card=15)'
STAT # 140430523695752 id=3 cnt=45 pid=2 pos=1 obj=0 op='NESTED LOOPS  (cr=7 pr=9 pw=0 time=42041 us cost=5 size=1995 card=15)'
STAT # 140430523695752 id=4 cnt=45 pid=3 pos=1 obj=0 op='NESTED LOOPS  (cr=5 pr=5 pw=0 time=40180 us cost=5 size=1995 card=40)'
STAT # 140430523695752 id=5 cnt=1 pid=4 pos=1 obj=0 op='NESTED LOOPS  (cr=4 pr=4 pw=0 time=30076 us cost=3 size=268 card=4)'
STAT # 140430523695752 id=6 cnt=1 pid=5 pos=1 obj=84729 op='TABLE ACCESS BY INDEX ROWID LOCATIONS (cr=2 pr=2 pw=0 time=27964 us cost=2 size=48 card=1)'
STAT # 140430523695752 id=7 cnt=1 pid=6 pos=1 obj=84752 op='INDEX RANGE SCAN LOC_CITY_IX (cr=1 pr=1 pw=0 time=19682 us cost=1 size=0 card=1)'
STAT # 140430523695752 id=8 cnt=1 pid=5 pos=2 obj=84732 op='TABLE ACCESS BY INDEX ROWID DEPARTMENTS (cr=2 pr=2 pw=0 time=2081 us cost=1 size=76 card=4)'
STAT # 140430523695752 id=9 cnt=1 pid=8 pos=1 obj=84748 op='INDEX RANGE SCAN DEPT_LOCATION_IX (cr=1 pr=1 pw=0 time=407 us cost=0 size=0 card=4)'
STAT # 140430523695752 id=10 cnt=45 pid=4 pos=2 obj=84744 op='INDEX RANGE SCAN EMP_DEPARTMENT_IX (cr=1 pr=1 pw=0 time=9811 us cost=0 size=0 card=10)'
STAT # 140430523695752 id=11 cnt=45 pid=3 pos=2 obj=84737 op='TABLE ACCESS BY INDEX ROWID EMPLOYEES (cr=2 pr=4 pw=0 time=1779 us cost=1 size=264 card=4)'
STAT # 140430523695752 id=12 cnt=19 pid=2 pos=2 obj=84735 op='TABLE ACCESS FULL JOBS (cr=6 pr=6 pw=0 time=1272 us cost=3 size=513 card=19)'
STAT # 140430523695752 id=13 cnt=107 pid=1 pos=2 obj=0 op='VIEW  index$_join$_002 (cr=9 pr=12 pw=0 time=18317 us cost=2 size=1284 card=107)'
STAT # 140430523695752 id=14 cnt=107 pid=13 pos=1 obj=0 op='HASH JOIN  (cr=9 pr=12 pw=0 time=17668 us)'
STAT # 140430523695752 id=15 cnt=107 pid=14 pos=1 obj=84739 op='INDEX FAST FULL SCAN EMP_EMP_ID_PK (cr=3 pr=6 pw=0 time=2049 us cost=1 size=1284 card=107)'
STAT # 140430523695752 id=16 cnt=107 pid=14 pos=2 obj=84747 op='INDEX FAST FULL SCAN EMP_NAME_IX (cr=6 pr=6 pw=0 time=13597 us cost=1 size=1284 card=107)'
```
wait表示等待事件信息，含有等待时间，操作时间(ela,单位为微秒)，等待事件的p1，p2，p3可用v$event_name获得，像上例中的等待事件'db file sequential read'，p1=file# 8，p2=block# ，p3=blocks，即表示该操作读取的是数据文件id号为8的文件，从p2块号开始，读取了几个块。  
fetch中，p和cr分别表示物理读和一致性读引起的buffer get，cu表示current read引起的buffer get，r表示处理的行数。  
stat表示在执行过程中消耗的资源信息统计，从stat可以划出执行计划步骤，正常的执行计划图：
```
------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name              | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                   |    15 |  2580 |    10   (0)| 00:00:01 |
|*  1 |  HASH JOIN                       |                   |    15 |  2580 |    10   (0)| 00:00:01 |
|*  2 |   HASH JOIN                      |                   |    15 |  2400 |     8   (0)| 00:00:01 |
|   3 |    NESTED LOOPS                  |                   |    15 |  1995 |     5   (0)| 00:00:01 |
|   4 |     NESTED LOOPS                 |                   |    40 |  1995 |     5   (0)| 00:00:01 |
|   5 |      NESTED LOOPS                |                   |     4 |   268 |     3   (0)| 00:00:01 |
|   6 |       TABLE ACCESS BY INDEX ROWID| LOCATIONS         |     1 |    48 |     2   (0)| 00:00:01 |
|*  7 |        INDEX RANGE SCAN          | LOC_CITY_IX       |     1 |       |     1   (0)| 00:00:01 |
|   8 |       TABLE ACCESS BY INDEX ROWID| DEPARTMENTS       |     4 |    76 |     1   (0)| 00:00:01 |
|*  9 |        INDEX RANGE SCAN          | DEPT_LOCATION_IX  |     4 |       |     0   (0)| 00:00:01 |
|* 10 |      INDEX RANGE SCAN            | EMP_DEPARTMENT_IX |    10 |       |     0   (0)| 00:00:01 |
|  11 |     TABLE ACCESS BY INDEX ROWID  | EMPLOYEES         |     4 |   264 |     1   (0)| 00:00:01 |
|  12 |    TABLE ACCESS FULL             | JOBS              |    19 |   513 |     3   (0)| 00:00:01 |
|  13 |   VIEW                           | index$_join$_002  |   107 |  1284 |     2   (0)| 00:00:01 |
|* 14 |    HASH JOIN                     |                   |       |       |            |          |
|  15 |     INDEX FAST FULL SCAN         | EMP_EMP_ID_PK     |   107 |  1284 |     1   (0)| 00:00:01 |
|  16 |     INDEX FAST FULL SCAN         | EMP_NAME_IX       |   107 |  1284 |     1   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------------
```
stat中的id对应就是plan table中的id，cnt表示返回的rows，在这里可以看到plan table中id=0只返回了15行，而10046返回的行数是真实执行返回的行数，plan table仅仅是根据表的统计信息(直方图，动态采样)而评估的返回结果。pid表示当前行源的父节点，pos表示同级行源的不同位置，一般同级行源先执行pos较小的操作。obj代表着dba_object.object_id，op对应plan table中的operation。
### 6. 10053事件
10053解析的是CBO为何选择该执行计划。
比如，CBO为何会选择index range scan？为何会选择table full scan？10053事件可以帮我们获取答案，10053仅有两个级别1和2，一般设置为1.
```
FUNG@linora> VARIABLE loc VARCHAR2(30)
FUNG@linora> EXEC :loc:='South San Francisco' 

PL/SQL procedure successfully completed.

FUNG@linora> alter session set events '10053 trace name context forever,level 1';

Session altered.

FUNG@linora> SELECT emp.last_name, emp.first_name, j.job_title, d.department_name, l.city,
  2  l.state_province, l.postal_code, l.street_address, emp.email,
  3  emp.phone_number, emp.hire_date, emp.salary, mgr.last_name
  4  FROM hr.employees emp, hr.employees mgr, hr.departments d, hr.locations l, hr.jobs j
  5  WHERE l.city=:loc
  6  AND emp.manager_id=mgr.employee_id
  7  AND emp.department_id=d.department_id
  8  AND d.location_id=l.location_id
  9  AND emp.job_id=j.job_id;
FUNG@linora> alter session set events '10053 trace name context off';        
```
10053事件的trace中，包含有查询相关表的信息，包括表、索引统计信息，直方图信息，Clustering Factor，单表访问路径，多表联合的方式，对于n表联合，它会采取多重方式进行评估，理论上需要进行(n-1)!次组合，但是CBO当发现下一次组合的cost比最优的要大时，它会停止分析下去。  
对于表信息统计如下
```
***************************************
BASE STATISTICAL INFORMATION
***********************
Table Stats::
  Table: JOBS  Alias:  J
    #Rows: 19  #Blks:  5  AvgRowLen:  33.00  ChainCnt:  0.00
  Column (# 1): JOB_ID(
    AvgLen: 8 NDV: 19 Nulls: 0 Density: 0.052632
Index Stats::
  Index: JOB_ID_PK  Col#: 1
    LVLS: 0  #LB: 1  #DK: 19  LB/K: 1.00  DB/K: 1.00  CLUF: 1.00
***********************
Table Stats::
  Table: DEPARTMENTS  Alias:  D
    #Rows: 27  #Blks:  5  AvgRowLen:  21.00  ChainCnt:  0.00
  Column (# 1): DEPARTMENT_ID(
    AvgLen: 4 NDV: 27 Nulls: 0 Density: 0.037037 Min: 10 Max: 270
  Column (# 4): LOCATION_ID(
    AvgLen: 3 NDV: 7 Nulls: 0 Density: 0.018519 Min: 1400 Max: 2700
    Histogram: Freq  #Bkts: 7  UncompBkts: 27  EndPtVals: 7
Index Stats::
  Index: DEPT_ID_PK  Col#: 1
    LVLS: 0  #LB: 1  #DK: 27  LB/K: 1.00  DB/K: 1.00  CLUF: 1.00
  Index: DEPT_LOCATION_IX  Col#: 4
    LVLS: 0  #LB: 1  #DK: 7  LB/K: 1.00  DB/K: 1.00  CLUF: 1.00
***********************
```
rows表示CBO对这张表行数的预估，栏位信息中，NDV表示number of distinct values，Density表示密度，在索引中，有个比较关键的CLUF，表示[Clustering Factor](/oracle/clustering-factor.html)。在D表的统计中，Column 4是含有Freq Histogram[Histogram](/oracle/oracle-histograms.html)。
```
Access path analysis for LOCATIONS
***************************************
SINGLE TABLE ACCESS PATH 
  Single Table Cardinality Estimation for LOCATIONS[L] 
  Column (# 4): CITY(
    AvgLen: 9 NDV: 23 Nulls: 0 Density: 0.043478
  Table: LOCATIONS  Alias: L
    Card: Original: 23.000000  Rounded: 1  Computed: 1.00  Non Adjusted: 1.00
  Access Path: TableScan
    Cost:  3.00  Resp: 3.00  Degree: 0
      Cost_io: 3.00  Cost_cpu: 41607
      Resp_io: 3.00  Resp_cpu: 41607
  Access Path: index (AllEqRange)
    Index: LOC_CITY_IX
    resc_io: 2.00  resc_cpu: 14673
    ix_sel: 0.043478  ix_sel_with_filters: 0.043478 
    Cost: 2.00  Resp: 2.00  Degree: 1
  Best:: AccessPath: IndexRange
  Index: LOC_CITY_IX
         Cost: 2.00  Degree: 1  Resp: 2.00  Card: 1.00  Bytes: 0
```
对L表单路径访问中，Card: Original: 23.000000 表示L表原始记录为23行，Rounded: 1表示预估输出结果，为1条记录。CBO认为可能使用以下2种方式访问T表：  
1.Access Path: TableScan  
2.Access Path: index (AllEqRange)  
其中，cost最低的为IndexRange。
```
***************************************
GENERAL PLANS
***************************************
Considering cardinality-based initial join order.
Permutations for Starting Table :0
Join order[1]:  LOCATIONS[L]# 0  JOBS[J]# 1  DEPARTMENTS[D]# 2  EMPLOYEES[EMP]# 3  EMPLOYEES[MGR]# 4
***********************
Best so far:  Table#: 0  cost: 2.0009  card: 1.0000  bytes: 48
              Table#: 1  cost: 5.0033  card: 19.0000  bytes: 1425
              Table#: 2  cost: 8.0440  card: 73.2857  bytes: 6862
              Table#: 3  cost: 11.0873  card: 15.1429  bytes: 2400
              Table#: 4  cost: 13.1681  card: 15.0013  bytes: 2580
Join order[2]:  LOCATIONS[L]# 0  JOBS[J]# 1  DEPARTMENTS[D]# 2  EMPLOYEES[MGR]# 4  EMPLOYEES[EMP]# 3
Join order aborted: cost > best plan cost--CBO停止计算
Join order[3]:  LOCATIONS[L]# 0  JOBS[J]# 1  EMPLOYEES[EMP]# 3  DEPARTMENTS[D]# 2  EMPLOYEES[MGR]# 4
Join order[8]:  LOCATIONS[L]# 0  DEPARTMENTS[D]# 2  EMPLOYEES[EMP]# 3  JOBS[J]# 1  EMPLOYEES[MGR]# 4
***********************
Best so far:  Table#: 0  cost: 2.0009  card: 1.0000  bytes: 48
              Table#: 2  cost: 3.0015  card: 3.8571  bytes: 268
              Table#: 3  cost: 4.6329  card: 15.1429  bytes: 1995
              Table#: 1  cost: 7.6730  card: 15.1429  bytes: 2400
              Table#: 4  cost: 9.7539  card: 15.0013  bytes: 2580
***************
Now joining: EMPLOYEES[EMP]# 3
***************
Best:: JoinMethod: NestedLoop
       Cost: 4.63  Degree: 1  Resp: 4.63  Card: 15.14 Bytes: 133
***************
Now joining: JOBS[J]# 1
***************
Best:: JoinMethod: Hash
       Cost: 7.67  Degree: 1  Resp: 7.67  Card: 15.14 Bytes: 160
***************
Now joining: EMPLOYEES[MGR]# 4
***************
Best:: JoinMethod: Hash
       Cost: 9.75  Degree: 1  Resp: 9.75  Card: 15.00 Bytes: 172
```
以上为表的连接顺序，当遇到cost值大于最优的，CBO就会停止计算，从上面可以看出，Join 2~7都比1的COST要大，8的比1的要小。在Join order[8]中，CBO计算出与各个表连接COST最小的连接方式，最后在执行计划中体现出来。  
### 7.CBO术语
#####Cardinality
Card是针对某条sql的某个具体执行步骤的执行结果所包含的记录的估算，在RAW 10046 trace文件中,STAT最后的card就表示每一个执行步骤结果集行数，而在10053 trace中，单表访问路径中包含了'Card: Original: 5.000000  Rounded: 5  Computed: 5.00  Non Adjusted: 5.00'，Orig表示CBD预估当前表总行数，Round则表示预估的返回结果四舍五入后的结果，Computed表示预估的返回结果。
```
# 10046 raw trace
STAT # 139958729041264 id=10 cnt=0 pid=1 pos=2 obj=84556 op='TABLE ACCESS BY INDEX ROWID DEPT (cr=0 pr=0 pw=0 time=0 us cost=1 size=13 card=1)'
# 10053 trace
Access path analysis for SALGRADE
***************************************
SINGLE TABLE ACCESS PATH
  Single Table Cardinality Estimation for SALGRADE[SALGRADE]
  Table: SALGRADE  Alias: SALGRADE
    Card: Original: 5.000000  Rounded: 5  Computed: 5.00  Non Adjusted: 5.00
  Access Path: TableScan
    Cost:  3.00  Resp: 3.00  Degree: 0
      Cost_io: 3.00  Cost_cpu: 36557
      Resp_io: 3.00  Resp_cpu: 36557
  Best:: AccessPath: TableScan
         Cost: 3.00  Degree: 1  Resp: 3.00  Card: 5.00  Bytes: 0
```
####Selectivity
Selectivity=有谓词条件的返回结果记录数/未添加谓词的返回结果记录数，从定义上知道，如果sel越小，Card就越小，CBO估算的成本也就小。对于sel和card有如下关系：
```
Computed Card=Original * Sel
```
####NDV
NDV=Number of distinct，表示表内不重复的值。如果目标列上既无直方图，也无NULL，则Sel=1/NDV。
To be continued!

Reference：  
[Maclean教你读Oracle 10046 SQL TRACE](http://www.askmaclean.com/archives/maclean-10046-sql-trace.html)  
[Maclean教你读SQL TRACE TKProf报告](http://www.askmaclean.com/archives/maclean-tech-tkprof-10046.html)
