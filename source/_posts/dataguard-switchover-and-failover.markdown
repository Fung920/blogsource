---
layout: post
title: "DataGuard主备切换及故障切换"
date: 2014-06-12 14:58:30
comments: false
categories: oracle
tags: dataguard
keywords: rac dataguard,swithover,failover
description: RAC DataGuard Switch over and Fail over,Oracle RAC DataGuard主备切换及故障切换
---
DataGuard是从Oracle7.3开始引进的，在8i之前叫standby database，9i后更名为DataGuard，是Oracle提供的灾备手段之一。
<!--more-->
### 1. 11g DataGuard新特性
10g的DataGuard提供了实时应用(real time apply)功能，通过使用Standby redo log，当日志被写入Standby Redo Log时，在standby会立刻应用这些redo，大大减少了故障发生时数据丢失的概率。
```
#standby开启实时应用
SQL> startup nomount
SQL> alter database mount standby database;
SQL> alter database recover managed standby database using current logfile disconnect;
SQL> Select recovery_mode from v$archive_dest_status;

RECOVERY_MODE
----------------------------------------------
MANAGED REAL TIME APPLY
```
11g提供active DataGuard(需要单独授权，如果没有授权，不应该使用它)功能，能在open read only下进行实时查询(real time query)，在10g中，查询standby跟redo apply只能两者选其一，因而有资源浪费嫌疑。11g的这个功能可大大提高的资源的利用率。查询事务在standby上，DML在Primary上。11g ADG配置可参照前文[Configure DataGuard in RAC](/configure-dataguard-in-rac.html)。
```
#开启实时查询
SQL> alter database recover managed standby database cancel;
SQL> alter database open;
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
SQL> select open_mode from v$database;

OPEN_MODE
----------------------------------------
READ ONLY WITH APPLY
#查询redo apply blocks是否增加，增加则表示在进行apply
#主库
SQL> select sequence#,status,thread#,block# from v$managed_standby;
 SEQUENCE# STATUS                      THREAD#     BLOCK#
---------- ------------------------ ---------- ----------
         0 CONNECTED                         0          0
        97 WRITING                           1        737
#备库
SQL> select sequence#,status,thread#,block# from v$managed_standby;
 SEQUENCE# STATUS                      THREAD#     BLOCK#
---------- ------------------------ ---------- ----------
         0 CONNECTED                         0          0
        73 APPLYING_LOG                      2        792
```
### 2. DataGuard三种保护模式
#### 2.1 最大保护(Maximum Protection)
这种模式提供最高级别数据丢失保护，以保证在Primary数据库故障时，零数据丢失。在这种模式下，日志传输必须是SYNC,LGWR,AFFIRM模式，在事务提交前，standby必须确认primary传送过来的redo被安全的写入至少一个standby database中的standby redo log，同时也要保证在primary也已经写好。如果primary无法往standby数据库的redo log中写日志，则primary会关闭。因此，在这种机制下，一般最大保护模式的standby都是设置多台，这样，单一standby故障并不会导致primary shutdown。
#### 2.2 最大性能(Maximum Performance)
这种模式是默认模式，表明主库的性能和可用性不受备库影响。且主库事务提交跟备库独立，虽然primary日志也一样至少写入一个可用的standby log，但是这种写可以是ASYNC的。且不要求使用standby redo log。
#### 2.3 最大可用性(Maximum Availability)
这种模式是前面两种模式的混合平衡体。即配置跟最大保护一样，但是primary往standby写日志遇到问题的时候，primary不会直接关闭，而是降级为最大性能模式，等standby修复好之后，自动切换为最大可用性。
三种保护模式对日志传输的要求如下：
[Minimum Requirements for Data Protection Modes](http://docs.oracle.com/cd/B19306_01/server.102/b14239/log_transport.htm#g1282139)
### 3.Switchover
<code>Switchover</code>表示事先已经计划好的主备库角色切换。例如减少维护停机时间。最常用的是迁移及升级。Switchover分以下几个步骤：  
1.告知主库即将要进行switchover。  
2.取消主库用户连接。  
3.产生特殊的日志记录，标记为End Of Redo(EOR)。  
4.转换主库为standby。  
5.一旦备库应用到最后的EOR，表明没有数据丢失，则转换备库为primary。  
### 4. Failover
<code>Failover</code>指的是意外导致的主备库角色切换，Failover后主备库DataGuard关系消失，需重新创建DataGuard。而switchover则保持主备关系，只不过角色切换过来而已。转换过程也跟switchover类似，只是因为是意外事件，所以主库是没机会写EOR redo记录的。  
Failover分为手动和自动，手动模式管理员完全控制Failover的角色切换，然后，手动模式需要人去探知，从而增长了业务停机时间。相反的，DataGuard的<b>Fast-Start Failover</b>特性自动检测失败信息，自动评估DataGuard配置信息，如果有需要，会自动选择指定的standby进行Failover，此功能需开启DataGuard Broker。根据保护模式不同，Failover存在丢失数据的可能。
### 5. 示例
```
ENV:
primary:11gr2 2 nodes rac ASM + Linux
standby:11gr2 1 node single ASM +Linux
```
#### 5.1 RAC DataGuard Switch over
#####查询主备库状态
```
SQL> set line 200
SQL> select inst_id,database_role,OPEN_MODE from  gv$database;
   INST_ID DATABASE_ROLE                    OPEN_MODE
---------- -------------------------------- ----------------------------------------
         2 PRIMARY                          READ WRITE
         1 PRIMARY                          READ WRITE
SQL> select inst_id,database_role,OPEN_MODE from  gv$database;
   INST_ID DATABASE_ROLE                    OPEN_MODE
---------- -------------------------------- ----------------------------------------
         1 PHYSICAL STANDBY                 READ ONLY WITH APPLY
```
#####主库检查switchover状态
如果是to standby或者sessions active表明可以进行切换，sessions active意味着主备库有活动sessions关联，在switchover前需要将这些sessions关闭(with session shutdown)。
```
SQL> select switchover_status from v$database;
SWITCHOVER_STATUS
----------------------------------------
TO STANDBY
```
#####主库操作
主库如果是RAC，则要求只保留其中一个节点，其他的关闭。
```
[oracle@fung01:/home/oracle]$ srvctl stop instance -d rac11g -i rac11g2
[oracle@fung01:/home/oracle]$ srvctl status database -d rac11g
Instance rac11g1 is running on node fung01
Instance rac11g2 is not running on node fung02
```
主库切换日志，并在主备端查看日志同步情况，对于open resetlogs后的数据库不适合用此语句查看，因为sequence会被重置，可以通过查看后台评估主备节点同步状态。
```
SQL> ALTER SYSTEM ARCHIVE LOG CURRENT;
System altered.
SQL> select thread#, max(sequence#)
  2  from v$archived_log a, v$database b
  3  where a.resetlogs_change# = b.resetlogs_change#
  4  group by thread# order by 1;

   THREAD# MAX(SEQUENCE#)
---------- --------------
         1            105
         2             79
```
#####主库切换
```
SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO STANDBY WITH SESSION SHUTDOWN;
Database altered.
```
#####备库切换
```
SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
Database altered.
SQL>  ALTER DATABASE OPEN;
Database altered.
```
#####后续工作
原主库mount到standby状态
```
SQL> shutdown immediate
SQL> startup mount
```
原主库停止远程归档路径
```
SQL> alter system set log_archive_dest_state_2=defer;
System altered.
```
原主库启用实时查询
```
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
Database altered.
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE cancel;
Database altered.
SQL> alter database open;
Database altered.
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
Database altered.
#后台日志
Fri Jun 13 10:54:31 2014
Media Recovery Log +DATA/arch/1_107_848925414.arc
Completed: ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT
Media Recovery Log +DATA/arch/1_108_848925414.arc
Media Recovery Log +DATA/arch/1_109_848925414.arc
Media Recovery Log +DATA/arch/1_110_848925414.arc
Media Recovery Waiting for thread 1 sequence 111 (in transit)
Recovery of Online Redo Log: Thread 1 Group 5 Seq 111 Reading mem 0
  Mem# 0: +DATA/rac11g/onlinelog/group_5.333.849348597
```
这时新主库状态为to standby,在新备库recovery前，其状态为FAILED DESTINATION，而备库状态为RECOVERY NEEDED。
```
SQL> select switchover_status from v$database;
SWITCHOVER_STATUS
----------------------------------------
TO STANDBY
```
开启备库另一个节点
```
[oracle@fung02:/home/oracle]$ srvctl start instance -d rac11g -i rac11g2
```
主库切换几次日志，主备库查看后台及日志同步信息。
```
SQL> alter system switch logfile;
System altered.
SQL> select thread#, max(sequence#)
  2  from v$archived_log a, v$database b
  3  where a.resetlogs_change# = b.resetlogs_change#
  4  group by thread# order by 1;

   THREAD# MAX(SEQUENCE#)
---------- --------------
         1            112
         2             79
```
#### 5.2 RAC DataGuard Failover
由于各种可能会导致主库无法开启，这里模拟一种最简单的模式，手动abort两个实例。这样，在RAC端，还是能mount数据库(故障类型最理想状态)，只要主库能启动到mount状态，那么Flush 就可以把没有发送的归档和current online redo 发送到备库，如果Flush成功，数据不会丢失。
```
#主库模拟故障，仅开启一个实例至mount状态
[oracle@fung02:/home/oracle]$ srvctl stop database -d rac11g -o abort
SQL> startup mount
ORACLE instance started.
SQL> select open_mode from v$database;

OPEN_MODE
----------------------------------------
MOUNTED
```
主库刷新日志
```
SQL> alter system flush redo to 'stdby';
System altered.
#主库后台日志明显有EOR信息
ARCH: End-Of-Redo Branch archival of thread 1 sequence 120
ARCH: LGWR is scheduled to archive destination LOG_ARCHIVE_DEST_2 after log switch
ARCH: Standby redo logfile selected for thread 1 sequence 120 for destination LOG_ARCHIVE_DEST_2
Flush End-Of-Redo Log thread 1 sequence 120
Archived Log entry 271 added for thread 1 sequence 120 ID 0x71630fe3 dest 1:
ARCH: Noswitch archival of thread 2, sequence 83
ARCH: End-Of-Redo Branch archival of thread 2 sequence 83
ARCH: LGWR is scheduled to archive destination LOG_ARCHIVE_DEST_2 after log switch
ARCH: Standby redo logfile selected for thread 2 sequence 83 for destination LOG_ARCHIVE_DEST_2
Flush End-Of-Redo Log thread 2 sequence 83
#备库后台日志
Standby switchover readiness check: Checking whether recoveryapplied all redo..
Database not available for switchover
  End-Of-REDO archived log file has not been recovered
  Incomplete recovery SCN:0:648156 archive SCN:0:670380
Physical Standby did not apply all the redo from the primary.
Fri Jun 13 12:01:38 2014
Resetting standby activation ID 1902317539 (0x71630fe3)
Standby switchover readiness check: Checking whether recoveryapplied all redo..
Physical Standby applied all the redo from the primary.
Media Recovery Waiting for thread 1 sequence 121
Fri Jun 13 12:01:39 2014
Standby switchover readiness check: Checking whether recoveryapplied all redo..
Physical Standby applied all the redo from the primary.
Standby switchover readiness check: Checking whether recoveryapplied all redo..
Physical Standby applied all the redo from the primary.
```
如果主库不能至mount状态，或者不是11g，上述步骤可以跳过，但有数据丢失的可能。
#####备库操作
查询gap，如果有，将对应的归档文件copy到备库，并registered。
```
SQL> select thread#, low_sequence#, high_sequence# from v$archive_gap;
no rows selected
#如果有gap，copy后注册
alter database register physical logfile '/arch/oradata/dg01/1_332800.dbf';
```
取消和停止应用
```
SQL> alter database recover managed standby database cancel;
Database altered.
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
Database altered.
#如果主库和备库之间的网络中断了，那么备库的RFS进程就会等待网络的连接，直到TCP超时。此时需要加上force关键字
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH force;
```
此时standby状态
```
SQL> select open_mode, switchover_status from v$database;
OPEN_MODE                                SWITCHOVER_STATUS
---------------------------------------- --------------------
READ ONLY                                TO PRIMARY
```
#####进行切换
```
SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
Database altered.
```
#####重启数据库
重启新主库，对外提供服务
```
SQL> shutdown immediate;
ORA-01109: database not open


Database dismounted.
ORACLE instance shut down.
SQL> startup
ORACLE instance started.

Total System Global Area  835104768 bytes
Fixed Size                  2257840 bytes
Variable Size             746589264 bytes
Database Buffers           79691776 bytes
Redo Buffers                6565888 bytes
Database mounted.
Database opened.
```
#####Failover后的还原
由于Failover后主备关系消失，是迫不得已的操作，当主库fix完后，需要重启还原，如果开启了flashback功能，则flashback至Failover时的SCN，再重新应用备库日志。或者重新创建standby，然后再switchover。
#####使用闪回还原主库
```
SQL> SELECT to_char(STANDBY_BECAME_PRIMARY_SCN) from V$DATABASE;
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
SQL> FLASHBACK DATABASE TO SCN &standby_became_primary_scn;
#将原主库转换成物理备库，并启动日志应用进程
SQL> ALTER DATABASE CONVERT TO PHYSICAL STANDBY;
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;
```
Reference:  
[Quick Guide to configuring Oracle 11gR2 Data Guard Physical Standby Part1](http://tinky2jed.files.wordpress.com/2011/03/configure-dataguard-11gr2-physical-standby-part-i.pdf)  
[Quick Guide to configuring Oracle 11gR2 Data Guard Physical Standby Part2](http://tinky2jed.files.wordpress.com/2011/05/configure-dataguard-11gr2-physical-standby-part-ii1.pdf)
