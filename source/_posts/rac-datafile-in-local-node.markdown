---
layout: post
title: "RAC Datafile in Local Node"
date: 2014-06-20 10:36:58
comments: false
categories: oracle
tags: rac
keywords: rac datafile in local node 
description: RAC数据文件错误地建立在本地
---
客户在创建数据文件时，误将ASM数据文件创建在本地磁盘，导致重启的时候另一个节点怎么也起不来。
<!--more-->
客户现场无法记录操作日志，本文用虚拟机模拟类似场景。环境为11.2.0.4双节点RAC，使用ASM存储。
#####节点1模拟创建表空间
```
SQL> create tablespace data datafile '/backup/rac11g/data01.dbf' size 100M autoextend off;

Tablespace altered.
SQL> alter user fung default tablespace data;

User altered.
SQL> col name for a50
SQL> select name,status from v$datafile;

NAME                                               STATUS
-------------------------------------------------- --------------
+DATA/rac11g/datafile/system.259.848925423         SYSTEM
+DATA/rac11g/datafile/sysaux.260.848925443         ONLINE
+DATA/rac11g/datafile/undotbs1.261.848925457       ONLINE
+DATA/rac11g/datafile/undotbs2.263.848925475       ONLINE
+DATA/rac11g/datafile/users.264.848925483          ONLINE
+DATA/rac11g/datafile/fung.298.849353657           ONLINE
+DATA/rac11g/datafile/data.286.849526463           ONLINE
/backup/rac11g/data01.dbf                          ONLINE
```
节点1创建文件没任何问题：
```
SQL> create table fung.obj as select * from user_objects;

Table created.
```
节点2创建会报错：
```
SQL> create table fung.obj2 as select * from fung.obj;
create table fung.obj2 as select * from fung.obj
                                             *
ERROR at line 1:
ORA-01157: cannot identify/lock data file 7 - see DBWR trace file
ORA-01110: data file 7: '/backup/rac11g/data01.dbf'
```
重启集群，会发现节点2起不来
```
[grid@fung01:/home/grid]$ srvctl stop database -d rac11g
[grid@fung01:/home/grid]$ srvctl start database -d rac11g
PRCR-1079 : Failed to start resource ora.rac11g.db
CRS-5017: The resource action "ora.rac11g.db start" encountered the following error: 
ORA-01157: cannot identify/lock data file 7 - see DBWR trace file
ORA-01110: data file 7: '/backup/rac11g/data01.dbf'
. For details refer to "(:CLSN00107:)" in
 "/u01/app/11gr2/grid/log/fung02/agent/crsd/oraagent_oracle/oraagent_oracle.log".

CRS-2674: Start of 'ora.rac11g.db' on 'fung02' failed
CRS-2632: There are no more servers to try to place resource 'ora.rac11g.db' 
          on that would satisfy its placement policy
```
#####解决方案
### 1. ASM CP
ASM CP命令是11g提供的，能将OS文件复制到ASM里面。而在10g中，使用<code>DBMS_FILE_TRANSFER</code>包进行处理。 DBMS_FILE_TRANSFER可以在同一台Oracle服务器上或两台Oracle 服务器之间复制文件。它使用directory对象来指定源directory和目的directory，因为directory支持ASM路径名称，所以DBMS_FILE_TRANSFER也支持ASM路径名。这使得从常规文件系统的ASM存储区移入和移出文件变得十分简单。
##### 11g ASM CP命令
```
ASMCMD> pwd
+data/rac11g/datafile
ASMCMD> ls
FUNG.298.849353657
SYSAUX.260.848925443
SYSTEM.259.848925423
UNDOTBS1.261.848925457
UNDOTBS2.263.848925475
USERS.264.848925483
ASMCMD> cp /backup/rac11g/data01.dbf ./
copying /backup/rac11g/data01.dbf -> +data/rac11g/datafile/data01.dbf
ASMCMD> ls
FUNG.298.849353657
SYSAUX.260.848925443
SYSTEM.259.848925423
UNDOTBS1.261.848925457
UNDOTBS2.263.848925475
USERS.264.848925483
data01.dbf
```
#####在节点1使用rename重命名该数据文件
```
SQL> alter tablespace data offline;
Tablespace altered.
SQL> alter database rename file '/backup/rac11g/data01.dbf' to '+data/rac11g/datafile/data01.dbf';
Database altered.
SQL> alter tablespace data online;
alter tablespace data online
*
ERROR at line 1:
ORA-01113: file 7 needs media recovery
ORA-01110: data file 7: '+DATA/rac11g/datafile/data01.dbf'

SQL> recover datafile 7;
Media recovery complete.
SQL> alter tablespace data online;

Tablespace altered.
#查看修改后的结果
SQL> select name,status from v$datafile;

NAME                                               STATUS
-------------------------------------------------- --------------
+DATA/rac11g/datafile/system.259.848925423         SYSTEM
+DATA/rac11g/datafile/sysaux.260.848925443         ONLINE
+DATA/rac11g/datafile/undotbs1.261.848925457       ONLINE
+DATA/rac11g/datafile/undotbs2.263.848925475       ONLINE
+DATA/rac11g/datafile/users.264.848925483          ONLINE
+DATA/rac11g/datafile/fung.298.849353657           ONLINE
+DATA/rac11g/datafile/data01.dbf                   ONLINE
```
#####开启节点2实例，同时观察实例后台日志是否正常
```
[oracle@fung02:/home/oracle]$ srvctl start instance -d rac11g -i rac11g2
```
### 2. RMAN copy功能
```
RMAN> sql "alter tablespace data offline";

using target database control file instead of recovery catalog
sql statement: alter tablespace data offline

RMAN> copy datafile '/backup/rac11g/data01.dbf' to '+data/rac11g/datafile/data01.dbf';

Starting backup at 2014-06-20 11:28:30
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=78 instance=rac11g1 device type=DISK
channel ORA_DISK_1: starting datafile copy
input datafile file number=00007 name=/backup/rac11g/data01.dbf
output file name=+DATA/rac11g/datafile/data01.dbf tag=TAG20140620T112831 RECID=20 STAMP=850735714
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:00:03
Finished backup at 2014-06-20 11:28:34
RMAN> sql "alter database rename file ''/backup/rac11g/data01.dbf'' 
to ''+data/rac11g/datafile/data01.dbf''";
SQL> alter tablespace data online;

Tablespace altered.
#查看修正结果
SQL> col tablespace_name for a10
SQL> col file_name for a50
SQL> set line 200
SQL> select tablespace_name,file_name,status,online_status from dba_data_files;

TABLESPACE FILE_NAME                                          STATUS             ONLINE_STATUS
---------- -------------------------------------------------- ------------------ --------------
SYSTEM     +DATA/rac11g/datafile/system.259.848925423         AVAILABLE          SYSTEM
SYSAUX     +DATA/rac11g/datafile/sysaux.260.848925443         AVAILABLE          ONLINE
UNDOTBS1   +DATA/rac11g/datafile/undotbs1.261.848925457       AVAILABLE          ONLINE
UNDOTBS2   +DATA/rac11g/datafile/undotbs2.263.848925475       AVAILABLE          ONLINE
USERS      +DATA/rac11g/datafile/users.264.848925483          AVAILABLE          ONLINE
FUNG       +DATA/rac11g/datafile/fung.298.849353657           AVAILABLE          ONLINE
DATA       +DATA/rac11g/datafile/data01.dbf                   AVAILABLE          ONLINE
```
### 3. 10g <code>DBMS_FILE_TRANSFER</code>
创建transfer所需directory：
```
SQL> create directory fs as '/backup/rac11g';
Directory created.
SQL> create directory asm as '+data/rac11g/datafile';
Directory created.
```
使用transfer包进行复制
```
SQL> alter tablespace data offline;
Tablespace altered.
SQL> exec dbms_file_transfer.copy_file('FS','data01.dbf','ASM','data01.dbf');

PL/SQL procedure successfully completed.
#asm磁盘组中查看是否有文件过来
ASMCMD [+data/rac11g/datafile] > ls
COPY_FILE.501.850736539
FUNG.298.849353657
SYSAUX.260.848925443
SYSTEM.259.848925423
UNDOTBS1.261.848925457
UNDOTBS2.263.848925475
USERS.264.848925483
data01.dbf
SQL> alter database rename file '/backup/rac11g/data01.dbf' to '+data/rac11g/datafile/data01.dbf';
Database altered.
SQL> alter tablespace data online;
Tablespace altered.
SQL> select tablespace_name,file_name,status,online_status from dba_data_files;

TABLESPACE FILE_NAME                                          STATUS             ONLINE_STATUS
---------- -------------------------------------------------- ------------------ --------------
SYSTEM     +DATA/rac11g/datafile/system.259.848925423         AVAILABLE          SYSTEM
SYSAUX     +DATA/rac11g/datafile/sysaux.260.848925443         AVAILABLE          ONLINE
UNDOTBS1   +DATA/rac11g/datafile/undotbs1.261.848925457       AVAILABLE          ONLINE
UNDOTBS2   +DATA/rac11g/datafile/undotbs2.263.848925475       AVAILABLE          ONLINE
USERS      +DATA/rac11g/datafile/users.264.848925483          AVAILABLE          ONLINE
FUNG       +DATA/rac11g/datafile/fung.298.849353657           AVAILABLE          ONLINE
DATA       +DATA/rac11g/datafile/data01.dbf                   AVAILABLE          ONLINE
```
#### 4. 总结
在对数据库增加数据文件或者表空间时，首先要确认这些数据文件是存放在本地还是ASM或者是裸设备中，不要盲目的自以为根目录下的/oradata就是数据文件存放地。总之身为一个DBA，任何情况下对生产系统的操作，都要小心谨慎，且在做任何操作前，有时间备份，先进行备份，绝对不做一个给别人留坑的DBA！
