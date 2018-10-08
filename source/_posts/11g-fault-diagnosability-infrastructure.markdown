---
layout: post
title: 11g新特性—FDI简介
categories:
- oracle
tags:
- new featrure
published: true
comments: false
date: 2013-09-09
---
 <h1>1.FDI简介</h1>
<p>从11g开始，Oracle增强了自动化错误诊断的功能。诊断数据包括以前版本的trace files，dumps，core file等等。</p> 
<!--more-->
<p>FDI（Fault Diagnosability Infrastructure）在于阻止、检测、诊断及解决问题。当数据库检测到critical errors，会将这些诊断数据保存到Automatic Diagnostic Repository(ADR)里面。ADR类似OFA，在诊断文件中，也有了系统的存储规划架构。</p> 
```
SQL> show parameter diagnostic_dest 

 NAME                                 TYPE        VALUE 
------------------------------------ ----------- ------------------------------ 
diagnostic_dest                      string      /u01/app/oracle 
SQL> !tree -d /u01/app/oracle/diag 
/u01/app/oracle/diag 
|-- asm 
|-- clients 
|-- crs 
|-- diagtool 
|-- lsnrctl 
|-- netcman 
|-- ofm 
|-- rdbms 
|   `-- racdb 
|       `-- racdb1 
|           |-- alert 
|           |-- cdump 
|           |-- hm 
|           |-- incident 
|           |-- incpkg 
|           |-- ir 
|           |-- lck 
|           |-- metadata 
|           |-- metadata_dgif 
|           |-- metadata_pv 
|           |-- stage 
|           |-- sweep 
|           `-- trace 
`-- tnslsnr 

 24 directories 

SQL> col name for a30 
SQL>  col value for a80 
SQL> set linesize 200 
SQL> select * from v$diag_info; 

    INST_ID NAME                           VALUE 
---------- ------------------------------ -------------------------------------------------------------------------------- 
         1 Diag Enabled                   TRUE 
         1 ADR Base                       /u01/app/oracle 
         1 ADR Home                       /u01/app/oracle/diag/rdbms/racdb/racdb1 
         1 Diag Trace                     /u01/app/oracle/diag/rdbms/racdb/racdb1/trace 
         1 Diag Alert                     /u01/app/oracle/diag/rdbms/racdb/racdb1/alert 
         1 Diag Incident                  /u01/app/oracle/diag/rdbms/racdb/racdb1/incident 
         1 Diag Cdump                     /u01/app/oracle/diag/rdbms/racdb/racdb1/cdump 
         1 Health Monitor                 /u01/app/oracle/diag/rdbms/racdb/racdb1/hm 
         1 Default Trace File             /u01/app/oracle/diag/rdbms/racdb/racdb1/trace/racdb1_ora_27059.trc 
         1 Active Problem Count           0 
         1 Active Incident Count          0 

 11 rows selected. 

 SQL>  
```
<h1>2.FDI核心组件</h1>
FDI的核心组件包括ADR、alter log、trace files，dumps，core files。
<h2>2.1. ADR</h2>
ADR是一个小型的外部XML数据库，它用于存储数据库，ASM，CRS等的诊断信息，每个实例拥有各自的ADR home目录，例如在一个RAC环境下，分别以grid及Oracle用户查询，将会得到不同的ADR HOME： 
```
[oracle@orcl1:/home/oracle]$ adrci 

 ADRCI: Release 11.2.0.4.0 - Production on Mon Sep 9 12:33:23 2013 

 Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved. 

 ADR base = "/u01/app/oracle" 
adrci> show home 
ADR Homes:  
diag/rdbms/racdb/racdb1 

 [grid@orcl1:/home/grid]$ adrci 

 ADRCI: Release 11.2.0.4.0 - Production on Mon Sep 9 12:33:40 2013 

 Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved. 

 ADR base = "/u01/app/grid" 
adrci> show home 
ADR Homes:  
diag/tnslsnr/orcl1/listener 
diag/asm/+asm/+ASM1 
adrci>  
```
因为从11g开始，所有的诊断文件，包括alter log都包含在ADR中，因此，BACKUPGROUND_DUMP_DEST及USER_DUMP_DESTl两个参数已经被废弃了，取而代之的是代表ADR目录的DIAGNOSTIC_DEST。如果此参数没有被设置，那么它将依靠以下两点进行设置默认值：
<ul>
	<li>如果ORACLE_BASE环境变量设置生效了，此参数将被设为ORACLE_BASE</li>
	<li>如果ORACLE_HOME没有设置，那么，此参数将会设置成ORACLE_HOME/log</li>
</ul>
<h2>2.2. ADRCI工具</h2>
在11g中的alter log已经是一个XML文件了，需要通过adrci工具或者EM工具才能查看，它包含了以下信息：
<ul>
	<li>严重错误、事件</li>
	<li>管理数据的一些动作，如启停数据库，恢复数据库或者创建删除表空间等</li>
	<li>自动刷新MView的错误</li>
	<li>其他数据库事件信息</li>
</ul>
<h2>2.3. ADRCI示例</h2>
首先模拟一个ora错误：
```
SQL> create undo tablespace undotbs2 datafile '/oradata/datafile/linora/undotbs02.dbf' size 1m;

Tablespace created.

SQL> 
SQL> alter system set undo_tablespace=undotbs2;

System altered.

SQL> show parameter undo;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_management                      string      AUTO
undo_retention                       integer     900
undo_tablespace                      string      UNDOTBS2
SQL> conn fung/oracle;
Connected.
SQL> create table test as select object_id,object_name from dba_objects;

Table created.

SQL> insert into test select * FROM TEST;
insert into test select * FROM TEST
            *
ERROR at line 1:
ORA-30036: unable to extend segment by 8 in undo tablespace 'UNDOTBS2'
```
切换至adrci命令行查找：
```
[root@linora ~]# su - oracle
[oracle@linora:/home/oracle]$ adrci

ADRCI: Release 11.2.0.4.0 - Production on Thu May 15 09:52:49 2014

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

ADR base = "/u01/app/oracle"
adrci> show problem

ADR Home = /u01/app/oracle/diag/tnslsnr/linora/listener:
*************************************************************************
0 rows fetched

ADR Home = /u01/app/oracle/diag/diagtool/user_oracle/host_1587426630_80:
*************************************************************************
PROBLEM_ID           PROBLEM_KEY                                                 LAST_INCIDENT        LASTINC_TIME                             
-------------------- ----------------------------------------------------------- -------------------- ---------------------------------------- 
1                    DIA 48001 [dbgvcis_ostream_write_1]                         1                    2014-05-15 09:26:41.240000 +08:00       

ADR Home = /u01/app/oracle/diag/rdbms/linora/linora:
*************************************************************************
0 rows fetched
```
上述结果显示了，在2014年5月15日，发生了一个dia-48001的错误。但这并不是一个rdbms的错误，因为ADR HOME目录是在diagtool下面。通过<code>show incident</code>可以查找出这个错误究竟影响了哪些东西：
```
adrci> show incident

ADR Home = /u01/app/oracle/diag/tnslsnr/linora/listener:
*************************************************************************
0 rows fetched

ADR Home = /u01/app/oracle/diag/diagtool/user_oracle/host_1587426630_80:
*************************************************************************
INCIDENT_ID          PROBLEM_KEY                                                 CREATE_TIME                              
-------------------- ----------------------------------------------------------- ---------------------------------------- 
1                    DIA 48001 [dbgvcis_ostream_write_1]                         2014-05-15 09:26:41.240000 +08:00       

ADR Home = /u01/app/oracle/diag/rdbms/linora/linora:
*************************************************************************
0 rows fetched
```
在本例中，并无影响。
通过以下命令，可以查找这个错误的trace file及trace file的详细信息：
```
adrci> show tracefile -I 1
     diag/diagtool/user_oracle/host_1587426630_80/incident/incdir_1/ora_2011_139919799314176_i1.trc
adrci> show incident -mode detail -p "incident_id=1" 

ADR Home = /u01/app/oracle/diag/tnslsnr/linora/listener:
*************************************************************************
0 rows fetched
<INCIDENT_INFO mode="detail">
<ADR_HOME name="/u01/app/oracle/diag/tnslsnr/linora/listener">

ADR Home = /u01/app/oracle/diag/diagtool/user_oracle/host_1587426630_80:
*************************************************************************

**********************************************************
INCIDENT INFO RECORD 1
**********************************************************
   INCIDENT_ID                   1
   STATUS                        ready
   CREATE_TIME                   2014-05-15 09:26:41.240000 +08:00
   PROBLEM_ID                    1
   CLOSE_TIME                    <NULL>
   FLOOD_CONTROLLED              none
   ERROR_FACILITY                DIA
   ERROR_NUMBER                  48001
   ERROR_ARG1                    dbgvcis_ostream_write_1
   ERROR_ARG2                    <NULL>
   ERROR_ARG3                    <NULL>
   ERROR_ARG4                    <NULL>
   ERROR_ARG5                    <NULL>
   ERROR_ARG6                    <NULL>
   ERROR_ARG7                    <NULL>
   ERROR_ARG8                    <NULL>
   ERROR_ARG9                    <NULL>
   ERROR_ARG10                   <NULL>
   ERROR_ARG11                   <NULL>
   ERROR_ARG12                   <NULL>
   SIGNALLING_COMPONENT          diag_fmwk
   SIGNALLING_SUBCOMPONENT       <NULL>
   SUSPECT_COMPONENT             <NULL>
   SUSPECT_SUBCOMPONENT          <NULL>
   ECID                          <NULL>
   IMPACTS                       0
   PROBLEM_KEY                   DIA 48001 [dbgvcis_ostream_write_1]
   FIRST_INCIDENT                1
   FIRSTINC_TIME                 2014-05-15 09:26:41.240000 +08:00
   LAST_INCIDENT                 1
   LASTINC_TIME                  2014-05-15 09:26:41.240000 +08:00
   IMPACT1                       0
   IMPACT2                       0
   IMPACT3                       0
   IMPACT4                       0
   OWNER_ID                      1
   INCIDENT_FILE                 /u01/app/oracle/diag/diagtool/user_oracle/host_1587426630_80/trace/ora_2011_139919799314176.trc
   OWNER_ID                      1
   INCIDENT_FILE                 /u01/app/oracle/diag/diagtool/user_oracle/host_1587426630_80/incident/incdir_1/ora_2011_139919799314176_i1.trc

ADR Home = /u01/app/oracle/diag/rdbms/linora/linora:
*************************************************************************
0 rows fetched
```
最后，可以通过<code>create package</code>对trace file进行打包，以便作为sr中的附件给oracle support分析。
```
adrci> set home diag/diagtool/user_oracle/host_1587426630_80
adrci> ips create package incident 1
Created package 1 based on incident id 1, correlation level typical
adrci> ips add incident 1 package 1
Added incident 1 to package 1
adrci> ips add file /u01/app/oracle/diag/rdbms/linora/linora/trace/alert_linora.log package 1
Added file /u01/app/oracle/diag/rdbms/linora/linora/trace/alert_linora.log to package 1
adrci> ips generate package 1 in /home/oracle
Generated package 1 in file /home/oracle/DIA48001d_20140515100720_COM_1.zip, mode complete
```
解压上述包，可用如下命令：
```
adrci> ips get metadata from file /home/oracle/DIA48001d_20140515100720_COM_1.zip
IPS metadata from file /home/oracle/DIA48001d_20140515100720_COM_1.zip:
----------------------------------------------------------
<?xml version="1.0" encoding="US-ASCII"?>
<PACKAGE>
    <PACKAGE_ID>1</PACKAGE_ID>
    <PACKAGE_NAME>DIA48001d_20140515100720</PACKAGE_NAME>
    <MODE>Complete</MODE>
    <SEQUENCE>1</SEQUENCE>
    <LAST_COMPLETE>1</LAST_COMPLETE>
    <DATE>2014-05-15 10:12:10.050452 +08:00</DATE>
    <ADR_BASE>/u01/app/oracle</ADR_BASE>
    <ADR_HOME>/u01/app/oracle/diag/diagtool/user_oracle/host_1587426630_80</ADR_HOME>
    <PROD_NAME>diagtool</PROD_NAME>
    <PROD_ID>user_oracle</PROD_ID>
    <INST_ID>host_1587426630_80</INST_ID>
    <OCM_GUID/>
    <OCM_ANNOTATION/>
    <FINALIZED>1</FINALIZED>
</PACKAGE>

----------------------------------------------------------
adrci>  ips unpack file /home/oracle/DIA48001d_20140515100720_COM_1.zip
Unpacking file /home/oracle/DIA48001d_20140515100720_COM_1.zip into current working directory
[oracle@linora ~]$ ls -l
total 102816
-rw-r--r-- 1 oracle oinstall    76421 May 15 10:12 DIA48001d_20140515100720_COM_1.zip
drwxrwxr-x 3 oracle oinstall     4096 May 15 10:20 diag
```
<h2>2.4. ADRCI基本命令</h2>
```
adrci> show base
ADR base is "/u01/app/oracle"
adrci> show home
ADR Homes: 
diag/tnslsnr/linora/listener
diag/diagtool/user_oracle/host_1587426630_80
diag/rdbms/linora/linora
adrci> set home diag/rdbms/linora/linora
adrci> show alert -tail 5
2014-05-15 09:51:15.569000 +08:00
ORA-1119 signalled during: create undo tablespace undotbs2 datafile '/oradata/datafile/linora/undotbs02.dbf'...
2014-05-15 09:51:47.709000 +08:00
create undo tablespace undotbs2 datafile '/oradata/datafile/linora/undotbs02.dbf' size 1m
ORA-1652: unable to extend temp segment by 8 in tablespace                 UNDOTBS2 
Completed: create undo tablespace undotbs2 datafile '/oradata/datafile/linora/undotbs02.dbf' size 1m
2014-05-15 09:51:55.397000 +08:00
[2048] Successfully onlined Undo Tablespace 6.
[2048] Undo Tablespace 2 successfully switched out.
ALTER SYSTEM SET undo_tablespace='UNDOTBS2' SCOPE=BOTH;
```
其他详细命令请参照手册：
<b>Oracle® Database Administrator's Guide 11<i>g</i> Release 2 (11.2)</b> 

 <b>EOF</b> 
