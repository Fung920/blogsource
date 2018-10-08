---
layout: post
title: "Oracle日志文件损坏的恢复"
date: 2014-06-05 10:06:11
comments: false
categories: oracle
tags: redo
keywords: redo,rman,recovery
description: how to hand types of redo log failures,Oracle日志文件损坏的恢复
---
如前文[管理redo文件](/redo-manage.html)所述，Online redo log是记录数据所有操作的地方，它最大的目的就是在数据库需要恢复的时候提供恢复的依据。
<!--more-->
Oracle的日志是循环写，因此，它要求至少要有两组日志文件，每个组至少有一个成员，成员就是磁盘中的物理文件。类似控制文件，Oracle也强烈要求日志文件多路存储在不同磁盘或者路径上。但由于日志文件没有一个很好的保护机制，它不能用RMAN进行备份，如果因为人为(rm -rf *.log)或者磁盘故障等导致日志文件损坏，该如何修复日志文件呢？  
1. 日志文件损坏信息都会被记录在alert log里面，因此，首先可以查看alert log确认是哪个日志的哪个成员损坏。  
2. 通过<code>v$log</code>和<code>v$logfile</code>，确认日志文件信息，这两个视图很有用。  
3. 如果仅仅是同一日志组的某一成员损坏，数据库仍会正常工作，只是在后台日志会报错。  
4. 日志文件状态决定了恢复策略。  
```
SQL> set linesize 200
SQL> col member for a50
SQL> select
a.group#, a.thread#,
a.status grp_status,
b.member member,
b.status mem_status
from v$log a,
v$logfile b
where a.group# = b.group#
order by a.group#, b.member;

    GROUP#    THREAD# GRP_STATUS       MEMBER                                             MEM_STA
---------- ---------- ---------------- -------------------------------------------------- -------
         1          1 CURRENT          /oracle/backup/redo01b.rdo                         INVALID
         1          1 CURRENT          /oracle/oradata/orcl/redo01.log
         2          1 INACTIVE         /oracle/backup/redo02b.rdo                         INVALID
         2          1 INACTIVE         /oracle/oradata/orcl/redo02.log
         3          1 INACTIVE         /oracle/backup/redo03b.rdo                         INVALID
         3          1 INACTIVE         /oracle/oradata/orcl/redo03.log
         5          1 UNUSED           /oracle/backup/redo05b.log
         5          1 UNUSED           /oracle/oradata/orcl/redo05a.log
```
<center>v$log视图status含义</center>

|Status|Meaning|
|----|----|
|CURRENT|正在使用的日志组|
|ACTIVE|当前日志文件需要在crash recovery用到，且可能尚未归档|
|CLEARING|正在被清除(alter database clear logfile)，清除完毕后，状态变为UNUSED|
|CLEARING_CURRENT|表明正在清除当前日志文件中的已关闭线程，如果切换时发生某些故障，如写入新日志标题时的I/O错误，则该日志可以停留在该状态|
|INACTIVE|表面此日志文件不需要在crash recovery用到，且可能尚未归档|
|UNUSED|表明从未对联机重做日志组进行写入，这种状态的日志文件要么而是刚增加的，要么是当日志不是current redo log时RESETLOGS操作后的状态|

<center>v$logfile视图status含义</center>

Status|Meaning
----|----
INVALID|日志文件无法被访问，或者是新增加的
DELETED|日志文件已经被删除，已不再使用
STALE|表明该文件内容不完全，例如正在添加一个日志文件成员
NULL|日志文件正在被使用

两个视图status栏位有不同的含义，v$log反应的是日志组的状态，v$logfile反应的是日志组成员即物理日志文件的状态。
### 1. 丢失多个成员中的一个
```
SQL> !rm -rf /oracle/backup/redo05b.log
Errors in file /oracle/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_6226036.trc:
ORA-00313: open failed for members of log group 5 of thread 1
ORA-00313: open failed for members of log group 5 of thread 1
ORA-00312: online log 5 thread 1: '/oracle/backup/redo05b.log'
ORA-27037: unable to obtain file status
IBM AIX RISC System/6000 Error: 2: No such file or directory
```
由于仅仅是丢失其中一份copy，因此数据库仍旧会正常工作，解决方法：
```
SQL> alter database drop logfile member '/oracle/backup/redo05b.log';
Database altered.
SQL> alter database add logfile member '/oracle/backup/redo05b.log' to group 5;
Database altered.
```
重新创建日志组成员可以在open或者mount状态下进行，但建议在mount下进行，这样能保证在drop和recreate过程中log group的状态不变。
### 2. 日志组状态inactive，丢失所有成员
```
SQL> !rm -rf /oracle/backup/redo05b.log
SQL> !rm -rf /oracle/oradata/orcl/redo05a.log
ORA-00312: online log 5 thread 1: '/oracle/oradata/orcl/redo05a.log'
ORA-27037: unable to obtain file status
IBM AIX RISC System/6000 Error: 2: No such file or directory
Additional information: 3
ORA-00313: open failed for members of log group 1 of thread 
ORA-00312: online log 5 thread 1: '/oracle/oradata/orcl/redo05a.log'
ORA-00312: online log 5 thread 1: '/oracle/backup/redo05b.log'
SQL> startup mount
SQL> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         46
         5 INACTIVE         YES          1         47
         3 INACTIVE         YES          1         45
         2 CURRENT          NO           1         48
```
确认损坏的日志组status为inactive，因为inactive状态的日志不需要在crash recovery中用到，因此，可以直接clear log方式恢复。如果尚未归档的，需要立刻对数据库进行备份。
```
SQL> alter database clear logfile group 5;
SQL> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         46
         2 CURRENT          NO           1         48
         3 INACTIVE         YES          1         45
         5 UNUSED           YES          1          0
```
对于尚未归档的日志组，需要用以下命令进行clear：
```
SQL> alter database clear unarchived logfile group 5;
```
### 3. 日志组状态为active，丢失所有成员
Active状态表示日志在crash recovery时候需要用到，且可能尚未归档。
```
SQL> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         46
         2 ACTIVE           YES          1         48
         3 INACTIVE         YES          1         45
         5 CURRENT          NO           1         49
[oracle@:/home/oracle]$ rm -rf /oracle/backup/redo02b.log
[oracle@:/home/oracle]$ rm -rf /oracle/oradata/orcl/redo02a.log
```
首先尝试进行Checkpoinnt
```
SQL> alter system checkpoint;
```
如果Checkpoinnt成功，那么，active状态的日志会变成inactive，Checkpoinnt成功后，所有被修改的脏块被写入磁盘，只有current状态的Online redo log才会被crash recovery用到。当日志组状态变为inactive后，clear log即可修复。同样，要是尚未归档，建议立刻进行备份。
```
SQL> alter database clear logfile group 2;

Database altered
SQL> select group#, status, archived, thread#, sequence# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 INACTIVE         YES          1         46
         2 UNUSED           YES          1          0
         3 INACTIVE         YES          1         45
         5 CURRENT          NO           1         49
```
对于尚未归档的日志组，需要用以下命令进行clear：
```
SQL> alter database clear unarchived logfile group 5;
```
### 4. 日志组状态为current，丢失所有成员
如果所有current状态日志组成员损坏，则需要不完全恢复，或是使用flashback database功能闪回。如果是DataGuard，还可以进行Failover切换。  
在准备不完全恢复前，先通过查询v$log的first_change#栏位确定能恢复的SCN值。但是，只能恢复到这个SCN前，而不包含当前SCN。
```
SQL> select group#, status, archived, thread#, sequence#, first_change# from v$log;

    GROUP# STATUS           ARC    THREAD#  SEQUENCE# FIRST_CHANGE#
---------- ---------------- --- ---------- ---------- -------------
         1 INACTIVE         YES          1         46        439571
         2 CURRENT          NO           1         50        472169
         3 INACTIVE         YES          1         45        403581
         5 UNUSED           YES          1          0        471475
[oracle@:/home/oracle]$ rm -rf /oracle/backup/redo02b.log
[oracle@:/home/oracle]$ rm -rf /oracle/oradata/orcl/redo02a.log
SQL> startup
ORACLE instance started.

Total System Global Area 1570009088 bytes
Fixed Size                  2221840 bytes
Variable Size            1006635248 bytes
Database Buffers          553648128 bytes
Redo Buffers                7503872 bytes
Database mounted.
ORA-03113: end-of-file on communication channel
Process ID: 10158148
Session ID: 63 Serial number: 5
#alert log
ORA-00313: open failed for members of log group 1 of thread 
ORA-00312: online log 2 thread 1: '/oracle/oradata/orcl/redo02.log'
ORA-00312: online log 2 thread 1: '/oracle/backup/redo02b.rdo'
```
在此例中，能恢复到SCN=472169前的数据，但不包含SCN=472169。
```
run{
allocate channel c1 type disk;
set until scn=472169;
restore database;
recover database;
release channel c1;
}
SQL> alter database open RESETLOGS;
SQL> col member for a50
SQL> set line 200
SQL> select a.group#, a.thread#,a.status grp_status,
  2  b.member member,b.status mem_status
  3  from v$log a,v$logfile b
  4  where a.group# = b.group#
  5  order by a.group#, b.member;

    GROUP#    THREAD# GRP_STATUS       MEMBER                                             MEM_STA
---------- ---------- ---------------- -------------------------------------------------- -------
         1          1 CURRENT          /oracle/backup/redo01b.rdo
         1          1 CURRENT          /oracle/oradata/orcl/redo01.log
         2          1 UNUSED           /oracle/backup/redo02b.rdo
         2          1 UNUSED           /oracle/oradata/orcl/redo02.log
         3          1 UNUSED           /oracle/backup/redo03b.rdo
         3          1 UNUSED           /oracle/oradata/orcl/redo03.log
         5          1 UNUSED           /oracle/backup/redo05b.log
         5          1 UNUSED           /oracle/oradata/orcl/redo05a.log
```

<b>EOF</b>  
Reference： RMAN recipes for oracle database 11g 








