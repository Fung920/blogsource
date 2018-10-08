---
layout: post
title: "Oracle Errorstack"
date: 2014-08-28 10:31:35
comments: false
categories: oracle
tags: 
keywords: errorstack
description: Oracle errorstack简介
---
Error stack经常用于一些错误排除和debug，它可用来追踪当前进程的状态，以确定问题根源所在。
<!--more-->
### 1. Error stack级别
Error stack包含四种级别，解释分别如下:

level|description
----|----
0|Error stack only
1|Error stack and function call stack
2|As level 1 + process state
3|As level 2 + context area

### 2. 追踪方法
Error stack可以在session级别和system级别进行追踪，同时还可以在spfile或者pfile中写入以下内容进行追踪：
```
event='1401 trace name errorstack, level 3'
```
#### 2.1 使用SQLPLUS追踪
```
SYS@linora>  create tablespace test datafile '/oradata/datafile/linora/test01.dbf' 
size 1024K autoextend off; 
--启动Error stack跟踪
FUNG@linora> alter session set events '1652 trace name errorstack level 3';
Session altered.
--模拟1652事件
FUNG@linora> create table error_stack tablespace test as select * from dba_objects;
create table error_stack tablespace test as select * from dba_objects
*
ERROR at line 1:
ORA-01652: unable to extend temp segment by 8 in tablespace TEST
--关闭跟踪
FUNG@linora> alter session set events='1652 trace name errorstack off';
Session altered.
FUNG@linora> @trace
VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_2430.trc
```
#### 2.2 使用oradebug追踪
oradebug使用需要sysdba的权限，对于Error stack的用法如下：
```
--设置追踪session为本session
SYS@linora> oradebug setmypid
Statement processed.
SYS@linora> oradebug event 942 trace name errorstack level 3;
Statement processed.
SYS@linora> select * from t;
select * from t
              *
ERROR at line 1:
ORA-00942: table or view does not exist

SYS@linora> oradebug event 942 trace name errorstack off;
Statement processed.
SYS@linora> oradebug tracefile_name
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_2523.trc
```
其他session追踪：
```
--找出需要追踪的session
SYS@linora> SELECT  a.sid,b.spid,a.username
  2  FROM   v$session a,v$process b
  3  WHERE   a.TYPE != 'BACKGROUND' and a.paddr=b.addr;
       SID SPID       USERNAME
---------- ---------- --------
       263 2652
       256 2654
        18 2583       SYS
       257 2632       FUNG
--以spid为例，如果是sid，则oradebug setorapid 257
SYS@linora> oradebug setospid 2632
Oracle pid: 33, Unix process pid: 2632, image: oracle@linora (TNS V1-V3)
SYS@linora> oradebug event 942 trace name errorstack level 3;
Statement processed.
SYS@linora> oradebug event 942 trace name errorstack off;
Statement processed.
SYS@linora> oradebug tracefile_name
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_2632.trc
```
