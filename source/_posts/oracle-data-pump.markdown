---
layout: post
title: "Oracle Data Pump"
date: 2014-05-20 11:14:35
comments: false
categories: oracle
tags: datapump
keywords: oracle data pump
description: oracle data pump utilities
---
<h3>1. Basic</h3>
Oracle Data Pump是10g以后用于数据逻辑导入导出的加强工具，加强是相对于<code>exp</code>跟<code>imp</code>而言。经常用于逻辑备份及数据库迁移方面，本文就经常用到的方法做一个小的总结。不同于exp/imp，data pump使用的时候要先创建一个目录用于存放dump文件。
<!--more-->
```
[oracle@:/oracle]$ ls -lt
total 2580304
drwxr-xr-x    3 oracle   oinstall        256 May 20 14:08 oradata
drwxr-xr-x    2 oracle   oinstall        256 May 20 14:07 arch
drwxr-xr-x    2 oracle   oinstall        256 May 20 14:07 expdir
```
10g默认的目录如下：
```
SQL> col OWNER for a10
SQL> col DIRECTORY_NAME for a20
SQL> col DIRECTORY_PATH for a50
SQL> set linesize 200
SQL> select * from dba_directories;

OWNER      DIRECTORY_NAME       DIRECTORY_PATH
---------- -------------------- --------------------------------------------------
SYS        DATA_PUMP_DIR        /oracle/app/oracle/product/10gr2/rdbms/log/
```
创建一个新的目录，并赋予Public的权限，这样，凡是有<code>exdpd</code>权限的用户均可读写此目录。
```
SQL> create directory expdir as '/oracle/expdir';

Directory created.
SQL> grant read,write on directory expdir to public;

Grant succeeded.

SQL> select * from dba_directories;

OWNER      DIRECTORY_NAME       DIRECTORY_PATH
---------- -------------------- --------------------------------------------------
SYS        DATA_PUMP_DIR        /oracle/app/oracle/product/10gr2/rdbms/log/
SYS        EXPDIR               /oracle/expdir
```
Data Pump是基于server端的，因此，目录也只能是基于server端。目录创建好，就可以进行导入导出动作了。  
Data Pump可基于表、用户、数据库等方式的导入导出。
<h4>1.1 Tables</h4>
```
[oracle@:/home/oracle]$ expdp fung/oracle@oratest tables=object directory=expdir dumpfile=fung.object.dmp logfile=fung.object.log

Export: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 14:52:16

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Enterprise Edition Release 10.2.0.1.0 - 64bit Production
With the Partitioning, OLAP and Data Mining options
Starting "FUNG"."SYS_EXPORT_TABLE_01":  fung/********@oratest tables=object directory=expdir dumpfile=fung.object.dmp logfile=fung.object.log 
Estimate in progress using BLOCKS method...
Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 32 MB
Processing object type TABLE_EXPORT/TABLE/TABLE
. . exported "FUNG"."OBJECT"                             26.63 MB  315008 rows
Master table "FUNG"."SYS_EXPORT_TABLE_01" successfully loaded/unloaded
******************************************************************************
Dump file set for FUNG.SYS_EXPORT_TABLE_01 is:
  /oracle/expdir/fung.object.dmp
Job "FUNG"."SYS_EXPORT_TABLE_01" successfully completed at 14:52:26

[oracle@:/home/oracle]$ impdp fung/oracle@oratest tables=object directory=expdir dumpfile=fung.object.dmp logfile=imp.object.log

Import: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 14:54:35

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Enterprise Edition Release 10.2.0.1.0 - 64bit Production
With the Partitioning, OLAP and Data Mining options
Master table "FUNG"."SYS_IMPORT_TABLE_01" successfully loaded/unloaded
Starting "FUNG"."SYS_IMPORT_TABLE_01":  fung/********@oratest tables=object directory=expdir dumpfile=fung.object.dmp logfile=imp.object.log 
Processing object type TABLE_EXPORT/TABLE/TABLE
Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
. . imported "FUNG"."OBJECT"                             26.63 MB  315008 rows
Job "FUNG"."SYS_IMPORT_TABLE_01" successfully completed at 14:54:40
```
<h4>1.2 Schemas</h4>
```
[oracle@:/home/oracle]$ expdp system/oracle schemas=fung directory=expdir dumpfile=fung.dmp logfile=fung.exp.log

Export: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 18:00:39

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Enterprise Edition Release 10.2.0.1.0 - 64bit Production
With the Partitioning, OLAP and Data Mining options
Starting "SYSTEM"."SYS_EXPORT_SCHEMA_01":  system/******** schemas=fung directory=expdir dumpfile=fung.dmp logfile=fung.exp.log 
Estimate in progress using BLOCKS method...
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 32 MB
Processing object type SCHEMA_EXPORT/USER
Processing object type SCHEMA_EXPORT/SYSTEM_GRANT
Processing object type SCHEMA_EXPORT/ROLE_GRANT
Processing object type SCHEMA_EXPORT/DEFAULT_ROLE
Processing object type SCHEMA_EXPORT/TABLESPACE_QUOTA
Processing object type SCHEMA_EXPORT/PRE_SCHEMA/PROCACT_SCHEMA
Processing object type SCHEMA_EXPORT/TABLE/TABLE
. . exported "FUNG"."OBJECT"                             26.63 MB  315008 rows
Master table "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully loaded/unloaded
******************************************************************************
Dump file set for SYSTEM.SYS_EXPORT_SCHEMA_01 is:
  /oracle/expdir/fung.dmp
Job "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully completed at 18:00:49

[oracle@:/home/oracle]$ impdp system/oracle@orcl schemas=fung directory=expdir dumpfile=oratest.dmp

Import: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 17:49:57

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
Master table "SYSTEM"."SYS_IMPORT_SCHEMA_01" successfully loaded/unloaded
Starting "SYSTEM"."SYS_IMPORT_SCHEMA_01":  system/********@orcl schemas=fung directory=expdir dumpfile=oratest.dmp 
Processing object type DATABASE_EXPORT/SCHEMA/USER
Processing object type DATABASE_EXPORT/SCHEMA/GRANT/SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/ROLE_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/DEFAULT_ROLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLESPACE_QUOTA
Processing object type DATABASE_EXPORT/SCHEMA/PROCACT_SCHEMA
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
. . imported "FUNG"."OBJECT"                             26.63 MB  315008 rows
Job "SYSTEM"."SYS_IMPORT_SCHEMA_01" successfully completed at 04:46:29
```
<h4>1.3 Database</h4>
```
[oracle@:/home/oracle]$ expdp system/oracle full=y directory=expdir dumpfile=oratest01.dmp
[oracle@:/home/oracle]$ impdp system/oracle@orcl full=y directory=expdir dumpfile=oratest.dmp
```
<h3>2. Advanced Options</h3>
<h4>2.1 exclude/include</h4>
使用<code>exclude</code>及<code>include</code>参数排除/加入某些表、用户等。  
语法：
```
EXCLUDE = object_type[:name_clause] [, ...]
INCLUDE = object_type[:name_clause] [, ...]
```
#####示例1
```
expdp hr/hr DIRECTORY=dpump_dir1 DUMPFILE=hr_exclude.dmp EXCLUDE=VIEW,PACKAGE, FUNCTION
```
#####示例2
全库导出，排除系统自带用户。
```
[oracle@:/home/oracle]$ expdp system/oracle full=y directory=expdir \
> EXCLUDE=SCHEMA:"IN ('OUTLN','DBSNMP','DIP')" \
> EXCLUDE=SCHEMA:"LIKE '%SYS%'" dumpfile=oratest.dmp

Export: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 18:11:12

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Enterprise Edition Release 10.2.0.1.0 - 64bit Production
With the Partitioning, OLAP and Data Mining options
ORA-39001: invalid argument value
ORA-39071: Value for EXCLUDE is badly formed.
ORA-00936: missing expression
```
出现这个错误，需要加上转义字符：
```
[oracle@:/home/oracle]$ expdp system/oracle full=y directory=expdir \
> EXCLUDE=SCHEMA:\"IN \(\'OUTLN\',\'DBSNMP\',\'DIP\'\)\" \
> EXCLUDE=SCHEMA:\"LIKE \'\%SYS\%\'\" dumpfile=oratest.dmp

Export: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 18:12:35

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Enterprise Edition Release 10.2.0.1.0 - 64bit Production
With the Partitioning, OLAP and Data Mining options
Starting "SYSTEM"."SYS_EXPORT_FULL_01":  system/******** full=y directory=expdir EXCLUDE=SCHEMA:"IN ('OUTLN','DBSNMP','DIP')" EXCLUDE=SCHEMA:"LIKE '%SYS%'" dumpfile=oratest.dmp 
Estimate in progress using BLOCKS method...
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 32 MB
Processing object type DATABASE_EXPORT/TABLESPACE
Processing object type DATABASE_EXPORT/SYS_USER/USER
Processing object type DATABASE_EXPORT/SCHEMA/USER
Processing object type DATABASE_EXPORT/ROLE
Processing object type DATABASE_EXPORT/GRANT/SYSTEM_GRANT/PROC_SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/GRANT/SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/ROLE_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/DEFAULT_ROLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLESPACE_QUOTA
Processing object type DATABASE_EXPORT/RESOURCE_COST
Processing object type DATABASE_EXPORT/TRUSTED_DB_LINK
Processing object type DATABASE_EXPORT/DIRECTORY/DIRECTORY
Processing object type DATABASE_EXPORT/DIRECTORY/GRANT/OWNER_GRANT/OBJECT_GRANT
Processing object type DATABASE_EXPORT/CONTEXT
Processing object type DATABASE_EXPORT/SYSTEM_PROCOBJACT/PRE_SYSTEM_ACTIONS/PROCACT_SYSTEM
Processing object type DATABASE_EXPORT/SYSTEM_PROCOBJACT/POST_SYSTEM_ACTIONS/PROCACT_SYSTEM
Processing object type DATABASE_EXPORT/SCHEMA/PROCACT_SCHEMA
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
Processing object type DATABASE_EXPORT/SCHEMA/POST_SCHEMA/PROCACT_SCHEMA
. . exported "FUNG"."OBJECT"                             26.63 MB  315008 rows
Master table "SYSTEM"."SYS_EXPORT_FULL_01" successfully loaded/unloaded
******************************************************************************
Dump file set for SYSTEM.SYS_EXPORT_FULL_01 is:
  /oracle/expdir/oratest.dmp
Job "SYSTEM"."SYS_EXPORT_FULL_01" successfully completed at 18:12:45
```
<h4>2.2 NETWORK_LINK</h4>
由于data pump是基于server端的，因此，所有的导入导出都是存储在server上。如果需要将远程数据库dump到本地来，必须要使用<code>network_link</code>参数。使用<code>network_link</code>要先创建dblink。
```
SQL> create public database link "orcl"
  2  connect to system
  3  identified by "oracle"
  4  using 'orcl';

Database link created.
[oracle@:/home/oracle]$ expdp system/oracle directory=expdir network_link=orcl dumpfile=netdmp.dmp
[oracle@:/home/oracle]$ impdp system/oracle directory=expdir schemas=fung network_link=orcl remap_schema=fung:kong
```
注意，使用<code>network_link</code>导入的时候，可以不产生dump文件，而直接导入目标数据库。
<h3>3. Miscellaneous </h3>
<h4>3.1 基于scn或者timestamp导入导出</h4>
```
#查询scn或者timestamp
SQL> SELECT current_scn FROM v$database;

CURRENT_SCN
-----------
     256822

SQL> SELECT DBMS_FLASHBACK.get_system_change_number FROM dual;

GET_SYSTEM_CHANGE_NUMBER
------------------------
                  256824

SQL> SELECT TIMESTAMP_TO_SCN(SYSTIMESTAMP) FROM dual;

TIMESTAMP_TO_SCN(SYSTIMESTAMP)
------------------------------
                        2
SQL> SELECT SCN_TO_TIMESTAMP(256826) FROM dual;

SCN_TO_TIMESTAMP(256826)
---------------------------------------------------------------------------
20-MAY-14 06.56.14.000000000 PM
```
过段时间，删除某些行。执行导出。
```
SQL> delete fung.object where rownum=1;

1 row deleted.

SQL> commit;

Commit complete.

SQL> select count(*) from object;

  COUNT(*)
----------
    315007

SQL> SELECT TIMESTAMP_TO_SCN(SYSTIMESTAMP) FROM dual;

TIMESTAMP_TO_SCN(SYSTIMESTAMP)
------------------------------
                        260057
[oracle@:/oracle/expdir]$ expdp system/oracle full=y directory=expdir dumpfile=full.emp flashback_scn=256826
```
删除表，执行导入，发现数据回复到表删除一行前的状态。
```
SQL> drop table fung.object;

Table dropped.
[oracle@:/oracle/expdir]$ impdp system/oracle directory=expdir dumpfile=full.emp schemas=fung

Import: Release 10.2.0.1.0 - 64bit Production on Tuesday, 20 May, 2014 19:13:20

Copyright (c) 2003, 2005, Oracle.  All rights reserved.

Connected to: Oracle Database 10g Enterprise Edition Release 10.2.0.1.0 - 64bit Production
With the Partitioning, OLAP and Data Mining options
Master table "SYSTEM"."SYS_IMPORT_SCHEMA_01" successfully loaded/unloaded
Starting "SYSTEM"."SYS_IMPORT_SCHEMA_01":  system/******** directory=expdir dumpfile=full.emp schemas=fung 
Processing object type DATABASE_EXPORT/SCHEMA/USER
ORA-31684: Object type USER:"FUNG" already exists
Processing object type DATABASE_EXPORT/SCHEMA/GRANT/SYSTEM_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/ROLE_GRANT
Processing object type DATABASE_EXPORT/SCHEMA/DEFAULT_ROLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLESPACE_QUOTA
Processing object type DATABASE_EXPORT/SCHEMA/PROCACT_SCHEMA
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE
Processing object type DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA
. . imported "FUNG"."OBJECT"                             26.63 MB  315008 rows
Job "SYSTEM"."SYS_IMPORT_SCHEMA_01" completed with 1 error(s) at 19:13:25
SQL> select count(*) from fung.object;

  COUNT(*)
----------
    315008
```
<h4>3.2 dump file中导出DDL语句</h4>
对于任意一个dump文件，<code>impdp</code>中，可用<code>sqlfile</code>将dump文件中的DDL语句单独导出到一个文本文件中。<code>sqlfile</code>参数是取代了<code>imp</code>中的<code>indexfile</code>参数。
```
impdp system/oracle schemas=fung directory=expdir sqlfile=fung.sql dumpfile=fung.dmp
```
<h3>4. wrap-up</h3>
1. 不同版本导入导出，需遵循一个原则：低版本导出，高版本导入。  
2. 在10.2.0.1版本中，不支持network_link导入导出11g，跟compatible参数有关，带确认。  
3. 全库导出/导入时，由于可能存在目标端数据库文件存储跟源端不一致，因此有可能导致表空间无法创建，对象无法创建的现象。在做全库导出时，排除掉系统用户，这样可以降低EM出错的概率，全库导入时，先创建表空间，再指定业务用户导入。  
4. data pump支持后台运行，如果导入导出过程中，不小心按掉ctrl+c，可以执行<code>expdp status</code>查看状态，并且删除job。  
5. expdp脚本
``` bash
#!/bin/sh
export HOME=/home/oracle
. $HOME/.profile
export DATE=`date '+%Y%m%d%H%M'`
export DMPFILE=$ORACLE_SID`date '+%Y%m%d%H%M'`_%U.dmp
export LOGFILE=$ORACLE_SID`date '+%Y%m%d%H%M'`.log
export EXPDIR=expdir
expdp system/oracle@linora schemas=fung,summer filesize=8192M \
directory=$EXPDIR dumpfile=$DMPFILE logfile=$LOGFILE parallel=2
exit 0
```

### 5.补充data pump参数文件使用
```
SCHEMAS=HR
DUMPFILE=expinclude.dmp
DIRECTORY=dpump_dir1
LOGFILE=expinclude.log
INCLUDE=TABLE:"IN ('EMPLOYEES', 'DEPARTMENTS')"
INCLUDE=PROCEDURE
INCLUDE=INDEX:"LIKE 'EMP%'"
```
以上为名为hr.par的参数文件，以下为用法
```
expdp hr/hr parfile=hr.par
```
Reference:  
[https://docs.oracle.com/cd/B19306_01/server.102/b14215/dp_export.htm#i1007837](https://docs.oracle.com/cd/B19306_01/server.102/b14215/dp_export.htm#i1007837) 


