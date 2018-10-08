---
layout: post
title: "Oracle修改数据库名称"
date: 2014-05-21 15:45:24
comments: false
categories: oracle
tags: nid
keywords: nid,change oracle database name and dbid
description: how to change database name or dbid in oracle by using nid and re-create controlfile,使用NID或者重建控制文件修改Oracle数据库名称
---
在DBA日常维护中，有时候会遇到需要修改数据库名称或者dbid的情况，可利用<code>nid</code>的工具对dbid和db_name进行修改，在某些情况下，还可以利用重建控制文件对db_name进行修改。
<!--more-->
<h3>1. nid工具</h3>
<h4>1.1 nid修改dbid</h4>
查询当前dbid
```
SQL> select dbid from v$database;

      DBID
----------
1375890310

SQL> 
```
<code>nid</code>工具语法
```
[oracle@:/home/oracle]$ nid help=y

DBNEWID: Release 11.2.0.3.0 - Production on Sat Jun 28 07:19:12 2014

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

Keyword     Description                    (Default)
----------------------------------------------------
TARGET      Username/Password              (NONE)
DBNAME      New database name              (NONE)
LOGFILE     Output Log                     (NONE)
REVERT      Revert failed change           NO
SETNAME     Set a new database name only   NO
APPEND      Append to output log           NO
HELP        Displays these messages        NO
```
启动数据库至mount状态：
```
SQL> startup mount pfile='/oracle/test/pfile.ora';
ORACLE instance started.

Total System Global Area 1570009088 bytes
Fixed Size                  2221840 bytes
Variable Size             922749168 bytes
Database Buffers          637534208 bytes
Redo Buffers                7503872 bytes
Database mounted.
SQL> select dbid,name,open_mode,activation#,created from v$database;

      DBID NAME      OPEN_MODE            ACTIVATION# CREATED
---------- --------- -------------------- ----------- ---------
1375890310 ORCLTEST  MOUNTED               2598928391 28-JUN-14
```
调用nid修改dbid
```
[oracle@:/oracle]$ nid TARGET=sys

DBNEWID: Release 11.2.0.3.0 - Production on Sat Jun 28 07:37:11 2014

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

Password: 
Connected to database ORCLTEST (DBID=1375890310)

Connected to server version 11.2.0

Control Files in database:
    /oracle/oradata/orcltest/control01.ctl
    /oracle/oradata/orcltest/control02.ctl

Change database ID of database ORCLTEST? (Y/[N]) => y

Proceeding with operation
Changing database ID from 1375890310 to 2598911656
    Control File /oracle/oradata/orcltest/control01.ctl - modified
    Control File /oracle/oradata/orcltest/control02.ctl - modified
    Datafile /oracle/oradata/orcltest/system01.db - dbid changed
    Datafile /oracle/oradata/orcltest/sysaux01.db - dbid changed
    Datafile /oracle/oradata/orcltest/undotbs01.db - dbid changed
    Datafile /oracle/oradata/orcltest/users01.db - dbid changed
    Control File /oracle/oradata/orcltest/control01.ctl - dbid changed
    Control File /oracle/oradata/orcltest/control02.ctl - dbid changed
    Instance shut down

Database ID for database ORCLTEST changed to 2598911656.
All previous backups and archived redo logs for this database are unusable.
Database has been shutdown, open database with RESETLOGS option.
Succesfully changed database ID.
DBNEWID - Completed succesfully.
```
查看dbid，发现已经改变。
```
SQL> startup
ORACLE instance started.

Total System Global Area 1570009088 bytes
Fixed Size                  2221840 bytes
Variable Size             922749168 bytes
Database Buffers          637534208 bytes
Redo Buffers                7503872 bytes
Database mounted.
ORA-01589: must use RESETLOGS or NORESETLOGS option for database open


SQL> alter database open resetlogs;

Database altered.

SQL> select dbid,name,open_mode,activation#,created from v$database;

      DBID NAME      OPEN_MODE            ACTIVATION# CREATED
---------- --------- -------------------- ----------- ---------
2598911656 ORCLTEST  READ WRITE            2598914511 28-JUN-14
```
注意，使用nid需要在mount状态下，且上一次关闭必须是干净的shutdown，否则会报<code>NID-00135: There are 1 active threads</code>的错误。
<h4>1.2 nid修改dbname</h4>
同样启动至mount状态进行修改
```
[oracle@:/home/oracle]$ nid TARGET=sys/oracle DBNAME=oratest SETNAME=yes

DBNEWID: Release 11.2.0.3.0 - Production on Sat Jun 28 07:43:44 2014

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

Connected to database ORCLTEST (DBID=2598911656)

Connected to server version 11.2.0

Control Files in database:
    /oracle/oradata/orcltest/control01.ctl
    /oracle/oradata/orcltest/control02.ctl

Change database name of database ORCLTEST to ORATEST? (Y/[N]) => y

Proceeding with operation
Changing database name from ORCLTEST to ORATEST
    Control File /oracle/oradata/orcltest/control01.ctl - modified
    Control File /oracle/oradata/orcltest/control02.ctl - modified
    Datafile /oracle/oradata/orcltest/system01.db - wrote new name
    Datafile /oracle/oradata/orcltest/sysaux01.db - wrote new name
    Datafile /oracle/oradata/orcltest/undotbs01.db - wrote new name
    Datafile /oracle/oradata/orcltest/users01.db - wrote new name
    Control File /oracle/oradata/orcltest/control01.ctl - wrote new name
    Control File /oracle/oradata/orcltest/control02.ctl - wrote new name
    Instance shut down

Database name changed to ORATEST.
Modify parameter file and generate a new password file before restarting.
Succesfully changed database name.
DBNEWID - Completed succesfully.
```
修改原有的pfile，将dbname改为目前的，至修改dbname，不需要resetlogs开启。
```
SQL> select dbid,name,open_mode,activation#,created from v$database;

      DBID NAME      OPEN_MODE            ACTIVATION# CREATED
---------- --------- -------------------- ----------- ---------
2598911656 ORATEST   READ WRITE            2598914511 28-JUN-14
```
如果需要同时修改dbname及dbid，则在<code>nid TARGET=sys/oracle DBNAME=oratest SETNAME=yes</code>中取消掉<code>setname=yes</code>即可。
<h3>2. 修改控制文件</h3>
通过修改控制文件去修改dbname，在这里模拟一种极端情况：7*24关键系统中，用户早上8点误删除数据，在下午5点发现，要求DBA恢复出8点误删除的数据，再没有多余的机器下，又不可能直接还原，只能通过备份还原到本机的不同目录，然后将丢失的数据重新插入。  
首先在生产库查找原有备份，并且恢复spfile到指定目录。
```
RMAN> list backup of spfile;

using target database control file instead of recovery catalog

List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
9       Full    7.42M      DISK        00:00:01     2014-06-28 07:03:01
        BP Key: 9   Status: AVAILABLE  Compressed: NO  Tag: HOT_DB_BK_LEVEL0
        Piece Name: /oracle/backup/bk_9_1_851410980_ORCL
  SPFILE Included: Modification time: 2014-06-28 02:46:09
  SPFILE db_unique_name: ORCL
RMAN> restore spfile to '/oracle/test/spfile.ora' from '/oracle/backup/bk_9_1_851410980_ORCL';

Starting restore at 2014-06-28 08:14:54
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=136 device type=DISK

channel ORA_DISK_1: restoring spfile from AUTOBACKUP /oracle/backup/bk_9_1_851410980_ORCL
channel ORA_DISK_1: SPFILE restore from AUTOBACKUP complete
Finished restore at 2014-06-28 08:14:55
[oracle@:/home/oracle]$ export ORACLE_SID=orcltest
SQL> create pfile from spfile='/oracle/test/spfile.ora';

File created.
```
使用vi编辑pfile，调整相应参数，包括db_name，控制文件存放路径及dump，adr存放目录，同时创建相关目录并且授权。  
从生产端备份控制文件至trace file。
```
SQL> alter database backup controlfile to trace as '/oracle/test/ctl.txt';

Database altered.
```
ctl.txt的关键内容
```
CREATE CONTROLFILE REUSE DATABASE "ORCL" RESETLOGS  ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/oracle/oradata/orcl/redo01.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 2 '/oracle/oradata/orcl/redo02.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 3 '/oracle/oradata/orcl/redo03.log'  SIZE 50M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '/oracle/oradata/orcl/system01.dbf',
  '/oracle/oradata/orcl/sysaux01.dbf',
  '/oracle/oradata/orcl/undotbs01.dbf',
  '/oracle/oradata/orcl/users01.dbf',
  '/oracle/oradata/orcl/fung01.dbf'
CHARACTER SET AL32UTF8
;
```
需要修改为如下内容
```
CREATE CONTROLFILE set DATABASE "ORCLTEST" RESETLOGS  ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/oracle/oradata/orcltest/redo01.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 2 '/oracle/oradata/orcltest/redo02.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 3 '/oracle/oradata/orcltest/redo03.log'  SIZE 50M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '/oracle/oradata/orcltest/system01.dbf',
  '/oracle/oradata/orcltest/sysaux01.dbf',
  '/oracle/oradata/orcltest/undotbs01.dbf',
  '/oracle/oradata/orcltest/users01.dbf',
  '/oracle/oradata/orcltest/fung01.dbf'
CHARACTER SET AL32UTF8
;
```
在执行创建语句之前，需要创建好相应的目录，并且利用rman在生产的SID上restore数据文件至相应目录，此例中为<code>/oracle/oradata/orcltest</code>。
```
RMAN> run{
2> allocate channel c1 type disk;
3> set newname for datafile 1 to '/oracle/oradata/orcltest/system01.dbf';
4> set newname for datafile 2 to '/oracle/oradata/orcltest/sysaux01.dbf';
5> set newname for datafile 3 to '/oracle/oradata/orcltest/undotbs01.dbf';
6> set newname for datafile 4 to '/oracle/oradata/orcltest/users01.dbf';
7> set newname for datafile 5 to '/oracle/oradata/orcltest/fung01.dbf';
8> restore database;
9> release channel c1;
10> }

allocated channel: c1
channel c1: SID=72 device type=DISK

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

Starting restore at 2014-06-28 08:30:41

channel c1: starting datafile backup set restore
channel c1: specifying datafile(s) to restore from backup set
channel c1: restoring datafile 00001 to /oracle/oradata/orcltest/system01.dbf
channel c1: restoring datafile 00002 to /oracle/oradata/orcltest/sysaux01.dbf
channel c1: restoring datafile 00003 to /oracle/oradata/orcltest/undotbs01.dbf
channel c1: restoring datafile 00004 to /oracle/oradata/orcltest/users01.dbf
channel c1: restoring datafile 00005 to /oracle/oradata/orcltest/fung01.dbf
channel c1: reading from backup piece /oracle/backup/bk_8_1_851410965_ORCL
channel c1: piece handle=/oracle/backup/bk_8_1_851410965_ORCL tag=HOT_DB_BK_LEVEL0
channel c1: restored backup piece 1
channel c1: restore complete, elapsed time: 00:00:25
Finished restore at 2014-06-28 08:31:06

released channel: c1
```
注意上述语句，不要执行<code>switch datafile</code>命令，因为我们仅仅需要还原数据文件，recover的动作交由后续创建好了的control file去完成。restore完后，在sid=orcltest上创建控制文件，之后进行恢复。
```
SQL> CREATE CONTROLFILE set DATABASE "ORCLTEST" RESETLOGS  ARCHIVELOG
  2      MAXLOGFILES 16
  3      MAXLOGMEMBERS 3
  4      MAXDATAFILES 100
  5      MAXINSTANCES 8
  6      MAXLOGHISTORY 292
  7  LOGFILE
  8    GROUP 1 '/oracle/oradata/orcltest/redo01.log'  SIZE 50M BLOCKSIZE 512,
  9    GROUP 2 '/oracle/oradata/orcltest/redo02.log'  SIZE 50M BLOCKSIZE 512,
 10    GROUP 3 '/oracle/oradata/orcltest/redo03.log'  SIZE 50M BLOCKSIZE 512
 11  -- STANDBY LOGFILE
 12  DATAFILE
 13    '/oracle/oradata/orcltest/system01.dbf',
 14    '/oracle/oradata/orcltest/sysaux01.dbf',
 15    '/oracle/oradata/orcltest/undotbs01.dbf',
 16    '/oracle/oradata/orcltest/users01.dbf',
 17    '/oracle/oradata/orcltest/fung01.dbf'
 18  CHARACTER SET AL32UTF8
 19  ;

Control file created.
```
创建完成后，数据库会自动进入mount状态，执行recover。
```
RMAN> recover database until time "to_date('2014-07-24 05:20:00','yyyy-mm-dd HH24:mi:ss')";
Starting recover at 2014-06-28 08:40:33
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 25 is already on disk as file /oracle/arch/1_25_848109574.arc
archived log for thread 1 with sequence 26 is already on disk as file /oracle/arch/1_26_848109574.arc
archived log for thread 1 with sequence 27 is already on disk as file /oracle/arch/1_27_848109574.arc
archived log for thread 1 with sequence 28 is already on disk as file /oracle/arch/1_28_848109574.arc
archived log for thread 1 with sequence 29 is already on disk as file /oracle/arch/1_29_848109574.arc
archived log for thread 1 with sequence 30 is already on disk as file /oracle/arch/1_30_848109574.arc
archived log for thread 1 with sequence 31 is already on disk as file /oracle/arch/1_31_848109574.arc
archived log for thread 1 with sequence 32 is already on disk as file /oracle/arch/1_32_848109574.arc
archived log for thread 1 with sequence 33 is already on disk as file /oracle/arch/1_33_848109574.arc
archived log for thread 1 with sequence 34 is already on disk as file /oracle/arch/1_34_848109574.arc
archived log file name=/oracle/arch/1_25_848109574.arc thread=1 sequence=25
archived log file name=/oracle/arch/1_26_848109574.arc thread=1 sequence=26
archived log file name=/oracle/arch/1_27_848109574.arc thread=1 sequence=27
archived log file name=/oracle/arch/1_28_848109574.arc thread=1 sequence=28
archived log file name=/oracle/arch/1_29_848109574.arc thread=1 sequence=29
archived log file name=/oracle/arch/1_30_848109574.arc thread=1 sequence=30
archived log file name=/oracle/arch/1_31_848109574.arc thread=1 sequence=31
archived log file name=/oracle/arch/1_32_848109574.arc thread=1 sequence=32
archived log file name=/oracle/arch/1_33_848109574.arc thread=1 sequence=33
archived log file name=/oracle/arch/1_34_848109574.arc thread=1 sequence=34
unable to find archived log
archived log thread=1 sequence=35
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 06/28/2014 08:40:36
RMAN-06054: media recovery requesting unknown archived log for thread 1 with sequence 35 and starting SCN of 328834
RMAN> sql 'alter database open resetlogs';

sql statement: alter database open resetlogs
SQL> select dbid,name,open_mode,activation#,created from v$database;

      DBID NAME      OPEN_MODE            ACTIVATION# CREATED
---------- --------- -------------------- ----------- ---------
1375890310 ORCLTEST  READ WRITE            2598924579 28-JUN-14
```
对于新库，需要的话可用以下命令创建密码文件。
```
orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID password=oracle entries=5 force=y
```
<b>在以上所有操作中，特别是生产环境下，oracle对所有open resetlogs的操作都有个建议：先备份数据库，在执行修改，修改完成后立即再次执行全备。</b>  
Reference:  
[ID 224266.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=224266.1)  
[ID 136183.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=136183.1)  
[ID 15390.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=15390.1)  
[ID 18070.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=18070.1)  
[Oracle® Database Utilities 11g Release 2 (11.2)](http://docs.oracle.com/cd/E11882_01/server.112/e22490/dbnewid.htm)
