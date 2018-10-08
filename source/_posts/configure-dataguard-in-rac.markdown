---
layout: post
title: "RAC DataGuard配置"
date: 2014-05-30
comments: false
categories: oracle
tags: dataguard
keywords: rac,dataguard,MAA
description: how to configure physical dataguard in 11gr2 rac,RAC数据库物理DataGuard
---
<h3>1.本文目的</h3>
DataGuard 是 Oracle 数据库的一个功能，能够提供数据库的冗余。冗余是通过创建一个备用（物理复制）数据库实现，备库最好是在不同的地理位置或者在不同的磁盘上。备库通过应用主库上的变化来保持数据同步。备库可以使用重做日志应用同步(物理备库)。物理备库只安装软件，不创建实例。  
<!--more-->
本文目的在于介绍Oracle 11gr2 RAC快速创建创建单机节点DataGuard的过程。
<h3>2.环境</h3>
```
主库：11gr2 RAC+ASM，DB_UNIQUE_NAME: rac11g  
备库：11gr2 Single+ASM/FS，DB_UNIQUE_NAME: stdby  
ORACLE_SID:rac11g  
Primary hostname：fung01，fung02  
Standby hostname：fung03  
Oracle数据库软件版本：  
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
操作系统版本均为OEL 5.8 x64 
```
DataGuard配置日志接收方式，本文采用LGWR。  
<h3>3.收集前期信息</h3>
IP规划
```
[grid@fung02:/home/grid]$ cat /etc/hosts
# Do not remove the following line, or various programs
# that require network functionality will fail.
127.0.0.1               fung01 localhost.localdomain localhost

#public IP 
192.168.192.101         fung01 
192.168.192.102         fung02

#priv 
10.10.10.101            fung01-priv 
10.10.10.102            fung02-priv

#Virtual IP 
192.168.192.103         fung01-vip 
192.168.192.104			fung02-vip

#SCAN 
192.168.192.105         rac11g-scan

#dataguard
192.168.192.107         fung03
```
主库日志文件
```
SQL>  select group#, thread#, bytes, blocksize, members from v$log;

    GROUP#    THREAD#      BYTES  BLOCKSIZE    MEMBERS
---------- ---------- ---------- ---------- ----------
         1          1   52428800        512          1
         2          1   52428800        512          1
         3          2   52428800        512          1
         4          2   52428800        512          1
SQL> set linesize 200
SQL> col member for a50
SQL> select member, group# from v$logfile order by 2;

MEMBER                                                 GROUP#
-------------------------------------------------- ----------
+DATA/rac11g/onlinelog/group_1.257.848925419                1
+DATA/rac11g/onlinelog/group_2.258.848925421                2
+DATA/rac11g/onlinelog/group_3.265.848926807                3
+DATA/rac11g/onlinelog/group_4.266.848926809                4
```
主库数据文件信息
```
SQL> col name for a80
SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
+DATA/rac11g/datafile/system.259.848925423
+DATA/rac11g/datafile/sysaux.260.848925443
+DATA/rac11g/datafile/undotbs1.261.848925457
+DATA/rac11g/datafile/undotbs2.263.848925475
+DATA/rac11g/datafile/users.264.848925483
```
服务名及其他
```
SQL> show parameter name

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
cell_offloadgroup_name               string
db_file_name_convert                 string
db_name                              string                 rac11g
db_unique_name                       string                 rac11g
global_names                         boolean                FALSE
instance_name                        string                 rac11g1
lock_name_space                      string
log_file_name_convert                string
processor_group_name                 string
service_names                        string                 rac11g
```
### 4.配置DataGuard
#### 4.1修改logging模式
备库要成为主库完全相同的复本，它必须接收来自主库的重做日志，Oracle数据库中，用户可以指定某些操作不产生日志(NOLOGGING子句)，对于备库来说，这是个很大的问题，因此，我们必须确认用户无法指示数据库不产生重做日志，即启用数据库的强制日志功能。
```
SQL> alter database force logging;

Database altered.

SQL>  select name, force_logging from v$database;

NAME                                                                             FORCE_
-------------------------------------------------------------------------------- ------
RAC11G                                                                           YES
```
#### 4.2为备库创建密码文件
```
[oracle@fung03:/home/oracle]$ orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID \
password=oracle entries=5 force=y ignorecase=y
[oracle@fung03:/home/oracle]$ cd -
/u01/app/oracle/product/11gr2/dbs
[oracle@fung03:/u01/app/oracle/product/11gr2/dbs]$ ls -lt
total 20
-rw-r----- 1 oracle oinstall 2048 May 30 15:15 orapwrac11g
-rw-rw---- 1 oracle oinstall 1544 May 30 15:05 hc_rac11g.dat
-rw-r--r-- 1 oracle oinstall 1229 May 30 15:05 initrac11g.ora
-rw-r----- 1 oracle oinstall 3584 May 30 15:04 spfilerac11g.ora
-rw-r--r-- 1 oracle oinstall 2851 May 15  2009 init.ora
```
#### 4.3更新网络配置文件
在创建备库之前，要确认两台服务器的数据库之间能通信，首先要配置主备库监听，使用RMAN的<code>duplicate</code>命令创建备库，备库必须首先处于NOMOUNT状态。在NOMOUNT状态下，数据库实例不会自动注册监听，必须配置静态监听。将以下信息添加进主库及备库的tnsnames.ora文档。
```
RAC11G =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = rac11g-scan)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = rac11g)
    )
  )

STDBY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = fung03)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = rac11g)
    )
  )
```
添加备库listener信息，静态注册备库listener.ora
```
[oracle@fung03:/u01/app/oracle/product/11gr2/network/admin]$ cat listener.ora 
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
       (GLOBAL_NAME= stdby)
       (ORACLE_HOME = /u01/app/oracle/product/11gr2)
       (SID_NAME = rac11g)
    )
  )
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = fung03)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC0))
    )
  )
```
同时启动备库监听
```
[oracle@fung03:/u01/app/oracle/product/11gr2/network/admin]$ lsnrctl start

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 30-MAY-2014 15:21:42

Copyright (c) 1991, 2013, Oracle.  All rights reserved.

Starting /u01/app/oracle/product/11gr2//bin/tnslsnr: please wait...

TNSLSNR for Linux: Version 11.2.0.4.0 - Production
System parameter file is /u01/app/oracle/product/11gr2/network/admin/listener.ora
Log messages written to /u01/app/oracle/diag/tnslsnr/fung03/listener/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=fung03)(PORT=1521)))
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC0)))

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=fung03)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 11.2.0.4.0 - Production
Start Date                30-MAY-2014 15:21:44
Uptime                    0 days 0 hr. 0 min. 1 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/11gr2/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/fung03/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=fung03)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC0)))
Services Summary...
Service "rac11g" has 1 instance(s).
  Instance "rac11g", status UNKNOWN, has 1 handler(s) for this service...
The command completed successfully
[oracle@fung03:/u01/app/oracle/product/11gr2/network/admin]$ lsnrctl service

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 30-MAY-2014 15:21:49

Copyright (c) 1991, 2013, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=fung03)(PORT=1521)))
Services Summary...
Service "rac11g" has 1 instance(s).
  Instance "rac11g", status UNKNOWN, has 1 handler(s) for this service...
    Handler(s):
      "DEDICATED" established:0 refused:0
         LOCAL SERVER
The command completed successfully
```
#### 4.4创建跟pfile中相同的文件目录
需要注意的是，这些路径要跟pfile中相对应。
```
[oracle@fung03:/home/oracle]$ mkdir -p /u01/app/oracle/diag/rdbms/rac11g/rac11g/trace
[oracle@fung03:/home/oracle]$ mkdir -p /u01/app/oracle/diag/rdbms/rac11g/rac11g/cdump
[oracle@fung03:/home/oracle]$ mkdir -p /u01/app/oracle/admin/rac11g/adump
```
### 5.创建DataGuard
#### 5.1配置主库DataGuard参数
```
#让主库知道DataGuard配置里的另一个库的名字
SQL> alter system set log_archive_config='dg_config=(rac11g,stdby)' scope=both;

System altered.
#配置主库本地归档路径
SQL> alter system set log_archive_dest_1='location=+DATA/arch valid_for=(all_logfiles,all_roles) db_unique_name=rac11g' scope=both;

System altered.

#配置主库远程归档路径
SQL> alter system set log_archive_dest_2='service=STDBY LGWR ASYNC NOAFFIRM valid_for=(online_logfiles,primary_role) db_unique_name=stdby' scope=both;

System altered.
以上归档切换时是用lgwr的异步传输方式
```
对于以上参数的简单说明：
`AFFIRM and NOAFFIRM`
AFFIRM：在写入到standby redo log 后，指定重做传输目标接受重做传输日志。  
NOAFFIRM：在写入到standby redo log前，重做传输日志可以传输到目的地。
`LOCATION and SERVICE`
LOCATION: 表示归档到本地。  
SERVICE: 表示归档到远程，跟tnsname.ora文件中的tnsname相同。
`SYNC and ASYNC`
指定使用同步还是异步传输模式。
`VALID_FOR`
指定数据库运行在主还是从数据库的角色。
是否online redo log files, standby redo log files 或是他们都将归档到该目的地，这里配置的是不管是online log还是standy log，也不管当前数据库是主库还是备库，都将归档到service=STDBY的远程归档上。
```
#开启归档路径1和归档路径2
SQL> alter system set log_archive_dest_state_1='enable' scope=both;

System altered.

SQL>  alter system set log_archive_dest_state_2='enable' scope=both;

System altered.

SQL> alter system set log_archive_max_processes=10 scope=both;

System altered.
```
添加备库自动管理文件功能，即当主库添加或删除数据文件时，这些文件也会在备库添加或删除。
```
SQL> alter system set standby_file_management='AUTO' scope=both;

System altered.
```
设置FAL_SERVER参数，此参数指定当日志传输出问题时，备库到哪里去找缺失的归档日志。它用在备库接收的到的重做日志间有gap的时候。这种情况会发生在日志传输出现中断时，如对备库需要维护而主库仍然正常运行，这时候，备库维护期间，没有日志传输过来，gap就出现了，设置了这个参数，备库会主动去寻找那些缺失的日志，并要求主库进行传输。
```
SQL> alter system set fal_server='stdby' scope=both;

System altered.
```
至此，主库DataGuard相关参数修改完毕，用以下语句确认是否有误。
```
SQL> set linesize 500 pages 0
SQL> col value for a120
SQL> col name for a50
SQL> select name, value
  2  from v$parameter
  3  where name in ('db_name','db_unique_name',
  4  'log_archive_config',
  5  'log_archive_dest_1','log_archive_dest_2',
  6  'log_archive_dest_state_1',
  7  'log_archive_dest_state_2',
  8  'remote_login_passwordfile',
  9  'log_archive_format',
 10  'log_archive_max_processes',
 11  'fal_server','db_file_name_convert',
 12  'log_file_name_convert',
 13  'standby_file_management')
 14  /
db_file_name_convert
log_file_name_convert
log_archive_dest_1                                 location=+DATA/arch valid_for=(all_logfiles,all_roles) db_unique_name=rac11g
log_archive_dest_2                                 service=STDBY LGWR ASYNC NOAFFIRM valid_for=(online_logfiles,primary_role) db_unique_name=stdby
log_archive_dest_state_1                           enable
log_archive_dest_state_2                           enable
fal_server                                         stdby
log_archive_config                                 dg_config=(rac11g,stdby)
log_archive_format                                 %t_%s_%r.arc
log_archive_max_processes                          10
standby_file_management                            AUTO
remote_login_passwordfile                          EXCLUSIVE
db_name                                            rac11g
db_unique_name                                     rac11g

14 rows selected.
```
### 5.2主库添加standby redo log
##### 添加原则
<li>standby redo log的文件大小与primary 数据库online redo log 文件大小相同</li>
<li>standby redo log日志文件组的个数依照下面的原则进行计算</li>
Standby redo log组数公式>=(每个instance日志组个数+1)\*instance个数，例如在我的环境中，有两个节点，每个节点有2组redo，所以Standby redo log组数公式>=(2+1)\*2  == 6，所以需要创建6组Standby redo log。
<li>每一日志组为了安全起见，可以包含多个成员文件</li>
```
alter database add standby logfile group 5 size 50M;
alter database add standby logfile group 6 size 50M;
alter database add standby logfile group 7 size 50M;
alter database add standby logfile group 8 size 50M;
alter database add standby logfile group 9 size 50M;
alter database add standby logfile group 10 size 50M;
```
查看结果
```
SQL> col member for a50
SQL> select member,type from v$logfile;

MEMBER                                             TYPE
-------------------------------------------------- --------------
+DATA/rac11g/onlinelog/group_1.257.848925419       ONLINE
+DATA/rac11g/onlinelog/group_2.258.848925421       ONLINE
+DATA/rac11g/onlinelog/group_3.265.848926807       ONLINE
+DATA/rac11g/onlinelog/group_4.266.848926809       ONLINE
+DATA/rac11g/onlinelog/group_5.333.849348597       STANDBY
+DATA/rac11g/onlinelog/group_6.334.849348599       STANDBY
+DATA/rac11g/onlinelog/group_7.335.849348601       STANDBY
+DATA/rac11g/onlinelog/group_8.336.849348603       STANDBY
+DATA/rac11g/onlinelog/group_9.337.849348605       STANDBY
+DATA/rac11g/onlinelog/group_10.338.849348607      STANDBY
```
### 6.编辑备库pfile
从主库创建pfile，取消RAC相关参数，并且添加以下参数：
```
*.db_name='rac11g'
*.db_unique_name='stdby'
*.standby_file_management='AUTO'
*.undo_tablespace='UNDOTBS1'
*.fal_server='rac11g'
*.log_archive_config='dg_config=(rac11g,stdby)'
*.log_archive_dest_1='location=/oradata/arch valid_for=(all_logfiles,all_roles) db_unique_name=stdby'
*.log_archive_dest_2='service=rac11g LGWR ASYNC NOAFFIRM valid_for=(online_logfiles,primary_role) db_unique_name=rac11g'
*.log_archive_dest_state_1='enable'
*.log_archive_dest_state_2='enable'
```
### 7.利用rman创建Standby
11g以后，可以利用duplicate active database或者从备份直接创建DataGuard，分别以两种不同方式创建，第一种以duplicate active standby还原至file system，第二种方式采用备份还原至ASM。  
先将备库启动至nomount模式。
#### 7.1duplicate active standby
由ASM还原至文件系统，需要经过set newname参数。同时因为日志文件路径也不同，需要在pfile中添加LOG_FILE_NAME_CONVERT参数进行变换，否则会报错:
```
RMAN-05535: WARNING: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-00344: unable to re-create online log '+data'
ORA-17502: ksfdcre:4 Failed to create file +data
ORA-15001: diskgroup "DATA" does not exist or is not mounted
ORA-15040: diskgroup is incomplete
ORA-15040: diskgroup is incomplete
```
执行复制脚本如下：
```
[oracle@fung03:/u01/app/oracle/product/11gr2/dbs]$ rman target sys/oracle@rac11g \
auxiliary sys/oracle@stdby msglog=/home/oracle/rac11g_`date '+%Y%m%d%H%M'`.log<<EOF
run{
allocate channel c1 type disk;
allocate auxiliary channel ch1 TYPE disk;
set newname for datafile 1  to '/oradata/rac11g/system01.dbf';
set newname for datafile 3  to '/oradata/rac11g/undotbs01.dbf';
set newname for datafile 2  to '/oradata/rac11g/sysaux01.dbf';
set newname for datafile 5  to '/oradata/rac11g/users01.dbf';
set newname for datafile 4  to '/oradata/rac11g/undotbs02.dbf';
set newname for tempfile 1  to '/oradata/rac11g/temp01.dbf';
DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE DORECOVER;
}
exit;
EOF
```
11g新特性，增加PARAMETER_VALUE_CONVERT参数，可参见[Duplicate Database With RMAN](/oracle/duplicate-database-with-rman.html)最后一点。
```
#For single to single
DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE
SPFILE
PARAMETER_VALUE_CONVERT '/oradata/datafile/linora/','/dup/'
SET "DB_UNIQUE_NAME"="stdby"
SET SGA_MAX_SIZE = '250M'
SET SGA_TARGET = '250M'
SET log_archive_dest_1 = 'LOCATION=/dup/arch/'
SET LOG_FILE_NAME_CONVERT='/oradata/datafile/linora/','/dup/'
DB_FILE_NAME_CONVERT='/oradata/datafile/linora/','/dup/';
#For rac to single
DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE
SPFILE
PARAMETER_VALUE_CONVERT 'rac11g','stdby'
SET "DB_UNIQUE_NAME"="stdby"
SET LOG_FILE_NAME_CONVERT='rac11g','stdby'
SET "CLUSTER_DATABASE"="FALSE"
SET "FAL_SERVER"="rac11g"
SET "FAL_CLIENT"="stdby"
SET "LOG_ARCHIVE_DEST_2"="service=rac11g LGWR ASYNC NOAFFIRM valid_for=(online_logfiles,primary_role) db_unique_name=rac11g"
SET "LOG_ARCHIVE_DEST_STATE_1"="enable"
SET "LOG_ARCHIVE_DEST_STATE_2"="enable"
RESET REMOTE_LISTENER
DB_FILE_NAME_CONVERT='rac11g','stdby';
```
#### 7.2使用备份创建DataGuard至ASM
11g ASM管理需要安装GI，然后把磁盘挂载即可。
```
SQL> select name,state,type,total_mb,free_mb
  2  from v$asm_diskgroup; 

NAME                           STATE       TYPE     TOTAL_MB    FREE_MB
------------------------------ ----------- ------ ---------- ----------
DATA                           MOUNTED     NORMAL      20480      20360
```
##### 创建主库rman备份
```
[oracle@fung01:/home/oracle]$ mkdir -p /worktmp/backup
[oracle@fung01:/home/oracle]$ rman target /
RMAN> backup database format '/worktmp/backup/bfull_%T_%u_%s_%p';
RMAN> backup current controlfile for standby format '/worktmp/backup/ctl_%T_%u_%s_%p';
RMAN> sql 'alter system archive log current';
RMAN> backup archivelog all format '/worktmp/backup/arc_%T_%u_%s_%p';
```
将备份集传送到备库相同目录，修改pfile，调整相关参数(主要是文件路径)，同时在ASM DATA磁盘组创建相关目录。并启动备库至nomount状态，rman中恢复备库
```
[oracle@fung03:/home/oracle]$ rman target sys/oracle@rac11g  auxiliary sys/oracle@stdby
RMAN> duplicate target database for standby nofilenamecheck dorecover;
```
在过程中遇到个问题：
```
RMAN> duplicate target database for standby nofilenamecheck dorecover;

Starting Duplicate Db at 2014-06-04 09:32:04
using target database control file instead of recovery catalog
allocated channel: ORA_AUX_DISK_1
channel ORA_AUX_DISK_1: SID=192 device type=DISK
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of Duplicate Db command at 06/04/2014 09:32:06
RMAN-05501: aborting duplication of target database
RMAN-06136: ORACLE error from auxiliary database: ORA-00200: control file could not be created
ORA-00202: control file: '+data'
ORA-17502: ksfdcre:4 Failed to create file +data
ORA-15001: diskgroup "DATA" does not exist or is not mounted
ORA-15040: diskgroup is incomplete
ORA-15040: diskgroup is incomplete
```
跟主库对比$ORACLE_HOME/bin/oracle的权限及asmdisk权限，发现都没错：
```
[oracle@fung03:/home/oracle]$ ls -la $ORACLE_HOME/bin/oracle
-rwxr-x--x 1 oracle oinstall 239627073 May 30 15:00 /u01/app/oracle/product/11gr2//bin/oracle
[oracle@fung01:/home/oracle]$ ls -la $ORACLE_HOME/bin/oracle
-rwsr-s--x 1 oracle asmadmin 239627031 May 30 12:17 /u01/app/oracle/product/11gr2//bin/oracle
[oracle@fung01:/home/oracle]$ ll /dev/asm*
brw-rw---- 1 grid asmadmin 8, 16 Jun  4 09:55 /dev/asm-diskb
brw-rw---- 1 grid asmadmin 8, 32 Jun  4 09:55 /dev/asm-diskc
brw-rw---- 1 grid asmadmin 8, 48 Jun  4 09:55 /dev/asm-diskd
brw-rw---- 1 grid asmadmin 8, 64 Jun  4 09:55 /dev/asm-diske
brw-rw---- 1 grid asmadmin 8, 80 Jun  4 09:55 /dev/asm-diskf
[oracle@fung03:/home/oracle]$ ll /dev/asm*
brw-rw---- 1 grid asmadmin 8, 64 Jun  4 09:51 /dev/asm-diske
brw-rw---- 1 grid asmadmin 8, 80 Jun  4 09:51 /dev/asm-diskf
[oracle@fung03:/home/oracle]$ id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(asmdba)
[oracle@fung03:/home/oracle]$ id grid
uid=54322(grid) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(asmdba),54324(asmadmin)
[oracle@fung01:/home/oracle]$ id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(asmdba)
[root@fung01 ~]# id grid
uid=54322(grid) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(asmdba),54324(asmadmin)
```
备库后台一直报错：
```
Additional information: 9
ORA-15025: could not open disk "/dev/asm-diskf"
ORA-27041: unable to open file
Linux-x86_64 Error: 13: Permission denied
Additional information: 9
SUCCESS: diskgroup DATA was dismounted
ERROR: diskgroup DATA was not mounted
```
发现是ORACLE_HOME/bin/oracle权限不对，解决方法:
```
To correct the proper group for the Database oracle executable, do:
As the <asm_home sfw owner>:
$ cd <asm_home>/bin
$ ./setasmgidwrap o=<db_home>/bin/oracle
```
再次执行duplicate动作，OK了。
### 8.备库开启实时应用
```
SQL> shutdown immediate;
SQL> startup nomount
SQL> alter database mount standby database;
SQL> alter database recover managed standby database using current logfile disconnect;
```
后台日志开始同步
```
Completed: alter database recover managed standby database using current logfile disconnect
Media Recovery Log +DATA/arch/1_65_848925414.arc
Media Recovery Log +DATA/arch/2_52_848925414.arc
Media Recovery Log +DATA/arch/2_53_848925414.arc
Media Recovery Log +DATA/arch/1_66_848925414.arc
Media Recovery Log +DATA/arch/2_54_848925414.arc
Media Recovery Log +DATA/arch/2_55_848925414.arc
Media Recovery Waiting for thread 2 sequence 56 (in transit)
Recovery of Online Redo Log: Thread 2 Group 6 Seq 56 Reading mem 0
  Mem# 0: +DATA/stdby/onlinelog/group_6.308.849351751
Media Recovery Waiting for thread 1 sequence 67 (in transit)
Recovery of Online Redo Log: Thread 1 Group 7 Seq 67 Reading mem 0
  Mem# 0: +DATA/stdby/onlinelog/group_7.307.849351753
```
主备库可用以下SQL检测log sequence是否一致：
```
select name,replace(database_role,' ','') as database_role,thread,seq 
from  v$database,(select max(sequence#) seq,THREAD# thread  
from v$log_history  group by THREAD#);
```
在主库添加数据文件，备库后台日志会同步相关信息
```
SQL> create tablespace fung datafile '+DATA' size 10M autoextend off;

Tablespace created.
#备库日志
Recovery created file +data
Successfully added datafile 6 to media recovery
Datafile # 6: '+DATA/stdby/datafile/fung.325.849353435'
```
### 9.为备库添加standby redo log
为备库添加standby log需要先停止MPR进程，添加完成后再重启MPR进程
```
SQL> alter database recover managed standby database cancel;

Database altered.

SQL>  alter database add standby logfile group 5 ('+DATA') size 50M;
SQL> alter database recover managed standby database using current logfile disconnect;

Database altered.
SQL> col member for a50
SQL> select member,type from v$logfile;

MEMBER                                             TYPE
-------------------------------------------------- --------------
+DATA/stdby/onlinelog/group_1.312.849351735        ONLINE
+DATA/stdby/onlinelog/group_2.311.849351739        ONLINE
+DATA/stdby/onlinelog/group_3.313.849351741        ONLINE
+DATA/stdby/onlinelog/group_4.310.849351745        ONLINE
+DATA/stdby/onlinelog/group_5.309.849351747        STANDBY
+DATA/stdby/onlinelog/group_6.308.849351751        STANDBY
+DATA/stdby/onlinelog/group_7.307.849351753        STANDBY
+DATA/stdby/onlinelog/group_8.314.849351757        STANDBY
+DATA/stdby/onlinelog/group_9.306.849351759        STANDBY
+DATA/stdby/onlinelog/group_10.300.849351763       STANDBY
```
### 10.验证DataGuard
验证日志是否从主库传输过来，最后一个栏位为yes表示日志已经传输过来
```
SELECT RESETLOGS_ID,THREAD#,SEQUENCE#,STATUS,ARCHIVED FROM V$ARCHIVED_LOG;
```
如果日志传输失败，请用以下命令查看主备库日志传输路径是否valid的
```
SQL> set line 200
SQL> col dest_name for a20
SQL> col error for a50
SQL> select dest_name,status,error from v$archive_dest where dest_name='LOG_ARCHIVE_DEST_2';
DEST_NAME            STATUS             ERROR
-------------------- ------------------ --------------------------------------------------
LOG_ARCHIVE_DEST_2   VALID
```
如果status为其他，则查看是什么原因导致无法归档到备库，调整完后用以下命令重启远程归档进程
```
SQL> alter system set log_archive_dest_state_2 = 'defer';
SQL> alter system set log_archive_dest_state_2 = 'enable';
```
备库查看lag 延时，正常所有lag应该接近0或者为0
```
SQL> col name for a30
SQL> col value for a30
SQL> col datum_time for a30
SQL> col TIME_COMPUTED for a20
SQL> set line 200
SQL> SELECT name, value, datum_time, time_computed FROM V$DATAGUARD_STATS;
NAME                           VALUE                          DATUM_TIME                     TIME_COMPUTED
------------------------------ ------------------------------ ------------------------------ --------------------
transport lag                  +00 00:00:00                   06/04/2014 11:43:01            06/04/2014 11:43:02
apply lag                      +00 00:00:00                   06/04/2014 11:43:01            06/04/2014 11:43:02
apply finish time              +00 00:00:00.000                                              06/04/2014 11:43:02
estimated startup time         33                                                            06/04/2014 11:43:02
```
查看DataGuard message
```
SQL> SELECT MESSAGE FROM V$DATAGUARD_STATUS;
MESSAGE
---------------------------------------------------------------------------
Media Recovery Waiting for thread 1 sequence 78 (in transit)
Media Recovery Waiting for thread 2 sequence 59 (in transit)
MRP0: Background Media Recovery cancelled with status 16037
Managed Standby Recovery not using Real Time Apply
MRP0: Background Media Recovery process shutdown
Managed Standby Recovery Canceled
Attempt to start background Managed Standby Recovery process
MRP0: Background Managed Standby Recovery process started
Managed Standby Recovery starting Real Time Apply
Media Recovery Waiting for thread 2 sequence 59 (in transit)
Media Recovery Waiting for thread 1 sequence 78 (in transit)
```
查看日志应用状态
```
SQL> select thread#,sequence#,applied from v$archived_log;
```
### 11.11gR2 DataGuard提供实时查询功能
##### 查看standby目前状态为mount状态
```
SQL> select open_mode from v$database;

OPEN_MODE
----------------------------------------
MOUNTED
```
##### 开启实时查询
```
#暂停MPR进程
SQL> alter database recover managed standby database cancel;
Database altered.
#开启real-time query
SQL> alter database open;
Database altered.
#开启MPR进程
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
Database altered.
```
##### 验证实时查询
```
#主库添加数据
SQL> create table t as select username from dba_users;
Table created.
#备库进行查询
SQL> select open_mode from v$database;

OPEN_MODE
----------------------------------------
READ ONLY WITH APPLY

SQL> select * from t;

USERNAME
------------------------------------------------------------
SYS
SYSTEM
OUTLN
APPQOSSYS
DBSNMP
WMSYS
DIP
ORACLE_OCM

8 rows selected.
```
### 12.删除standby归档脚本
```
grep "Media Recovery Log" /u01/app/oracle/diag/rdbms/stdby/rac11g/trace/alert_rac11g.log|sed -e 's/Media Recovery Log/rm -if/g' >/home/oracle/rman/dellog.sh
chmod u+x /home/oracle/rman/dellog.sh
sh  /home/oracle/rman/dellog.sh
cat /u01/app/oracle/diag/rdbms/stdby/rac11g/trace/alert_rac11g.log >>/u01/app/oracle/diag/rdbms/stdby/rac11g/trace/alert_rac11g_old.log
cat /dev/null > /u01/app/oracle/diag/rdbms/stdby/rac11g/trace/alert_rac11g.log
```
See more details in:  
[Oracle® Data Guard Concepts and Administration 11g Release 2 (11.2)](http://docs.oracle.com/cd/E11882_01/server.112/e41134/rcmbackp.htm#SBYDB01500)
