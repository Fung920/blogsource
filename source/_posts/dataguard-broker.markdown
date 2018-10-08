---
layout: post
title: "DataGuard Broker"
date: 2014-06-16 11:02:41
comments: false
categories: oracle
tags: dataguard
keywords: dataguard broker
description: dataguard broker configuration,dataguard broker配置
---
Dataguard broker是9i开始引进的，随同Oracle企业版(Enterprise Edtion,EE)跟DataGuard一起内置的管理监控工具。通过DataGuard Broker，DBA能简化部署、监控DataGuard，以及主备角色切换。
<!--more-->
### 1. Broker简介
Broker由三部分组成：主备库各自的Broker后台进程，一系列的配置文件和命令行工具<code>dgmgrl</code>。需要注意的是，如果使用Broker管理或配置DataGuard，则不应该使用SQL命令再进行配置管理，否则会导致Broker参数配置或者数据库配置不一致的情况。  
Broker处理流程如下图所示，这些进程是在数据库及Broker启动自动运行，DBA无法介入。  
![Broker Process](/images/dgbroker1.png)  
进程说明如下：  
<li><b>DataGuard Monitor(DMON)：</b>Broker主进程，所有进程中，最先启动DMON，它的主要任务是协调Broker的其他进程，包括维护Broker配置文件，这个进程通过<code>DG_BROKER_START</code>进行激活和取消。</li>
<li><b>Broker Resource Manager(RSM)：</b>RSM处理Broker在任何一个数据库中进行配置所执行的所有SQL命令。</li>
<li><b>DataGuard Net Server (NSVn)：</b>NSVn负责跟远程数据库连接。当创建配置文件时候需要指定连接符，NSVn就是通过这个连接符去连接远程数据库。</li>
<li><b>DRCn：</b>网络接收进程，负责建立与source端NSVn进程的联系。NSVn到DRCn类似于在日志传输时的LogWriter Network Service(LNS)到Remote File Server(RSF)的连接，当日志传输时，Broker需要发送数据或者SQL命令，会使用NSV及DRC进行连接，这些连接只会在需要的情况下启动。</li>
<li><b>Configuration files：</b>配置文件为在主库和备库上的常规二进制文件，它储存了DataGuard的所有参数配置。可以存储在OS文件系统上或者ASM存储上，它是通过Broker去进行维护，而不是认为去维护，类似SPIFLE。</li>
在Broker配置中，主库的DMON进程对DG配置有拥有权，不管在哪一个节点上对配置进行修改，都是在主节点完成。任何时候DMON需要执行一些SQL，它都会调用主库的RSM进行辅助。如果SQL执行目标是主库，它会直接执行；如果SQL执行目标是备库，RSM会要求NSVn进程将这些SQL命令传送到备库端，通过DRCn进程执行。  
使用Broker可通过Enterprises Manager Grid Control或者命令行工具<code>dgmgrl</code>。本文只介绍命令行工具的使用。dgmgrl工具是整合在EE版本中，包括Client端。我们甚至可以用Windows平台的dgmgrl去管理Linux平台的Broker配置，但一般不建议这么做。
### 2. Broker配置
配置Broker先决条件：  
<li>由于Broker在配置过程中需要动态调整数据库参数，因此，必须要使用SPFILE。</li>
<li>RAC环境中，配置文件必须存储在共享磁盘上，如ASM，通过调整<code>DG_BROKER_CONFIG_FILEn</code>指定配置文件存放路径。</li>
<li>主备库数据库版本必须相同。</li>
<li>为了DMON能自动启动，<code>DG_BROKER_START</code>参数必须设置为<code>TRUE</code>。</li>
#### 2.1 设置配置文件
```
#实验环境为11gR2 RAC + Single DataGuard
#主备库同时修改
[grid@fung02:/home/grid]$ asmcmd -p
ASMCMD [+] > cd data
ASMCMD [+data] > mkdir BROKER
SQL> ALTER SYSTEM SET DG_BROKER_CONFIG_FILE1 = '+DATA/BROKER/dr1rac11g.dat' scope=both SID='*';
System altered.
SQL> ALTER SYSTEM SET DG_BROKER_CONFIG_FILE2 = '+DATA/BROKER/dr2rac11g.dat' scope=both SID='*';
System altered.
SQL> select name,value from v$parameter where name like 'dg_broker_config_file%';
NAME                           VALUE
------------------------------ --------------------------------------------------
dg_broker_config_file1         +DATA/BROKER/dr1rac11g.dat
dg_broker_config_file2         +DATA/BROKER/dr2rac11g.dat

SQL> show parameter spfile
NAME                  TYPE                   VALUE
--------------------- ---------------------- ------------------------------
spfile                string                 +DATA/rac11g/spfilerac11g.ora
```
#### 2.2 Enable DMON
```
#主备库启动DMON
SQL> ALTER SYSTEM SET DG_BROKER_START=TRUE SCOPE=BOTH sid='*';
System altered.
SQL> SHOW PARAMETER DG_BROKER_START

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
dg_broker_start                      boolean                TRUE
```
顺序不能颠倒，必须要先设置好dg_broker_config_file，否则会报错：
```
SQL> ALTER SYSTEM SET dg_broker_config_file1 = '+DATA/mfg/dr1mfg.dat' scope=both sid='*'; 
ALTER SYSTEM SET dg_broker_config_file1 = '+DATA/mfg/dr1mfg.dat' scope=both sid='*'
*
ERROR at line 1:
ORA-02097: parameter cannot be modified because specified value is invalid
ORA-16573: attempt to change or access configuration file for an enabled broker configuration
```
#### 2.3 配置网络监听
Broker需要在监听中增加${ORACLE_SID}_DGMGRL.db_domain的条目。所以，在各自节点对主备库都进行静态监听。在主备库grid用户下，增加相关条目至listener.ora
```
#For node2, please change SID_NAME with node2s SID
#primary
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = rac11g_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/11gr2)
      (SID_NAME = rac11g1)
    )
  )
#standby
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_NAME = stdby_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/11gr2)
      (SID_NAME = rac11g)
    )
  )
```
主备库重启监听
```
[grid@fung01:/u01/app/11gr2/grid/network/admin]$ srvctl stop listener
[grid@fung01:/u01/app/11gr2/grid/network/admin]$ srvctl start listener
[grid@fung01:/u01/app/11gr2/grid/network/admin]$ lsnrctl status

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 17-JUN-2014 10:43:42

Copyright (c) 1991, 2013, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=LISTENER)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 11.2.0.4.0 - Production
Start Date                17-JUN-2014 10:43:36
Uptime                    0 days 0 hr. 0 min. 5 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/11gr2/grid/network/admin/listener.ora
Listener Log File         /u01/app/grid/diag/tnslsnr/fung01/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=LISTENER)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=127.0.0.1)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.192.103)(PORT=1521)))
Services Summary...
Service "rac11g" has 1 instance(s).
  Instance "rac11g1", status READY, has 1 handler(s) for this service...
Service "rac11g_DGMGRL" has 1 instance(s).
  Instance "rac11g1", status UNKNOWN, has 1 handler(s) for this service...
The command completed successfully
```
#### 2.4 配置Broker
```
[oracle@fung01:/home/oracle]$ dgmgrl /
DGMGRL for Linux: Version 11.2.0.4.0 - 64bit Production

Copyright (c) 2000, 2009, Oracle. All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected.
DGMGRL> create configuration drrac11g as
> primary database is RAC11G
> connect identifier is RAC11G;
Configuration "drrac11g" created with primary database "rac11g"
```
如果名称错了，可用<code>DGMGRL> remove configuration;</code>移除配置，上一步骤指定了主库，包括其所有实例。下一步指定备库到Broker。
```
DGMGRL> add database stdby as connect identifier is stdby maintained as physical;
Database "stdby" added
DGMGRL> enable configuration;
Enabled.
DGMGRL> show database STDBY

Database - stdby

  Role:            PHYSICAL STANDBY
  Intended State:  APPLY-ON
  Transport Lag:   0 seconds (computed 1 second ago)
  Apply Lag:       0 seconds (computed 1 second ago)
  Apply Rate:      79.00 KByte/s
  Real Time Query: ON
  Instance(s):
    rac11g

Database Status:
SUCCESS
DGMGRL> SHOW CONFIGURATION;

Configuration - drrac11g

  Protection Mode: MaxPerformance
  Databases:
    rac11g - Primary database
    stdby  - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
```
### 3. switchover
如果是RAC，无论主备库，只保留一个节点实例，其他的关闭：
```
[oracle@fung02:/home/oracle]$ srvctl stop instance -d rac11g -o immediate -i rac11g2
```
进行switchover切换
```
DGMGRL> connect sys/oracle@stdby
Connected.
DGMGRL> switchover to stdby
Performing switchover NOW, please wait...
New primary database "stdby" is opening...
Operation requires startup of instance "rac11g1" on database "rac11g"
Starting instance "rac11g1"...
ORACLE instance started.
Database mounted.
Database opened.
Switchover succeeded, new primary is "stdby"
```
不像手动切换一样，需要手动将另一个实例起来，dgmrl会自动启动node2的实例。
```
SQL> select inst_id,database_role,OPEN_MODE from  gv$database;

   INST_ID DATABASE_ROLE                    OPEN_MODE
---------- -------------------------------- ----------------------------------------
         2 PHYSICAL STANDBY                 READ ONLY WITH APPLY
         1 PHYSICAL STANDBY                 READ ONLY WITH APPLY

```
### 4. 配置Fast-Start Failover
Fast-Start Failover是建立在broker基础上的一个快速故障转换的机制，通过fast-start failover可以自动检测primary的故障，然后自动的failover到预先指定的standby上面，这样可以最大化的减少故障时间，提高数据库的可用性。Fast-Start Failover是在broker的基础上再增加了一个单独的observer server，用来监控primary和standby数据库的状态，一旦primary不可用，observer就会自动的切换到指定的standby上面。
```
DGMGRL>  show fast_start failover;
Fast-Start Failover: DISABLED
```
启动Fast-Start Failover需要先开启闪回数据库(Flashback Database).且DataGuard必须在最大可用性模式下才能进行Fast-Start Failover。
```
SQL> Alter system set db_recovery_file_dest_size=2G;
SQL> Alter system set db_recovery_file_dest='+DATA';
$srvctl stop database -d rac11g -o immediate
$srvctl start database -d rac11g -o mount
SQL> Alter database flashback on;
#备库如果在open状态也要先重启至mount状态，如果备库在mount状态，则要先取消redo apply
SQL> alter database recover managed standby database cancel;
SQL> Alter database flashback on;
SQL> alter database recover managed standby database disconnect from session;
```
#####修改DataGuard保护模式
```
#查询当前保护模式
SQL>select protection_mode,protection_level from v$database;
#修改日志接收方式
SQL> ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=stdby LGWR SYNC AFFIRM VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=stdby'; 
$srvctl stop database -d rac11g
$srvctl start database -d rac11g -o mount
SQL>alter database set standby database to maximize availability;
$srvctl start database -d rac11g -o open
#为了切换，同时修改standby日志接收参数
SQL> ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=rac11g OPTIONAL LGWR ASYNC AFFIRM VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=rac11g';
```
#####配置Fast-Start Failove参数
```
-- 配置rac11g failover的目标
dgmgrl> edit database rac11g set property 'FastStartFailoverTarget'='stdby';
-- 配置stdby failover的目标
dgmgrl> edit database stdby set property 'FastStartFailoverTarget'='rac11g';
--配置延时参数
dgmgrl> EDIT CONFIGURATION SET PROPERTY FastStartFailoverThreshold = 30;
DGMGRL> ENABLE FAST_START FAILOVER;
DGMGRL> start observer;
```



































