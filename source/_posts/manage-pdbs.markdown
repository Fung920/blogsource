---
layout: post
title: "12C之PDB管理"
date: 2014-08-13 10:00:41
comments: false
categories: oracle
tags: 12c
keywords: 12c pdb
description: how to manage pdbs in oracle 12c
---
在上一篇文章[12c Silent Installation](/12c-silent-installation.html)创建的一个空的CDB，本文介绍如何使用sqlplus工具对PDB进行维护(EM Express更简单，以下所有操作均可从EM生成对应的SQL)。
<!--more-->
### 1. 添加PDB
添加PDB前需要满足以下几个条件：
<li>数据库必须是CDB</li>
<li>CDB处于READ/WRITE模式</li>
<li>创建PDB的用户必须是common user</li>
<li>创建PDB的用户要有CREATE PLUGGABLE DATABASE的系统权限</li>
<li>每个PDB须有不同的名称</li>
在12.1中，一个CDB最多支持253个PDB，其中包括一个PDB$SEED。新增pdb的方法有以下几种：
![pdb create](/images/pdbcreate.png)
#### 1.1 使用seed创建新的pdb
![Create a PDB Using the Seed Files](/images/pdbseed.png)
这种方法复制seed中的文件到新的PDB。
```
CREATE PLUGGABLE DATABASE salespdb ADMIN USER salesadm IDENTIFIED BY oracle roles=(DBA)
  STORAGE (MAXSIZE 2G MAX_SHARED_TEMP_SIZE 100M)
  DEFAULT TABLESPACE sales 
    DATAFILE '/u02/oradata/ora12c/salespdb/sales01.dbf' SIZE 250M AUTOEXTEND ON
  PATH_PREFIX = '/u02/oradata/ora12c/salespdb/'
  FILE_NAME_CONVERT = ('/u02/oradata/ora12c/pdbseed/', '/u02/oradata/ora12c/salespdb/');
```
<li>Storage</li>
MAXSIZE定义一个属于这个PDB的表空间总容量；MAX_SHARED_TEMP_SIZE定义一个属于这个PDB的临时表空间总容量。如果设定为<code>storage unlimited</code>，或者没有设定此参数，均表示此PDB的没有存储限制。存储限制可于创建完PDB后修改：
```
sqlplus sys/oracle@linora:1522/salespdb as sydsba
SQL> alter pluggable database salespdb storage(maxsize 20G);
```
<li>File Location of the New PDB</li>
在新的PDB中，有两个子句可以在创建PDB的时候指定文件路径：<code>FILE_NAME_CONVERT</code>和<code>CREATE_FILE_DEST</code>，后一个子句是用于OMF管理。而<code>PATH_PREFIX</code>所有跟此PDB相关的文件路径都会被存在在此参数指定的路径下。
<li>ROLES</li>
给定管理者salsadm权限，这个用户是PDB local user，本例中预授的是PDB_DBA权限。  
创建完后可以看到PDB的状态：
```
SYS@ora12c> select con_id, name,open_mode from v$containers;
    CON_ID NAME       OPEN_MODE
---------- ---------- --------------------
         1 CDB$ROOT   READ WRITE
         2 PDB$SEED   READ ONLY
         3 SALESPDB   MOUNTED
```
开启PDB至OPEN状态：
```
SYS@ora12c> ALTER PLUGGABLE DATABASE salespdb open;
Pluggable database altered.
SYS@ora12c> col name for a15
SYS@ora12c> select con_id, name,open_mode from v$containers;
    CON_ID NAME            OPEN_MODE
---------- --------------- --------------------
         1 CDB$ROOT        READ WRITE
         2 PDB$SEED        READ ONLY
         3 SALESPDB        READ WRITE
```
#### 1.2 从本地PDB Clone
```
--clone时，source PDB必须处于read only状态
SYS@ora12c> alter pluggable database salespdb close;
Pluggable database altered.
SYS@ora12c> alter pluggable database salespdb open read only;
Pluggable database altered.
SYS@ora12c> CREATE PLUGGABLE DATABASE hrpdb FROM salespdb no data
  2  FILE_NAME_CONVERT = ('/u02/oradata/ora12c/salespdb/', '/u02/oradata/ora12c/hrpdb/')
  3  STORAGE unlimited;
Pluggable database created.
SYS@ora12c> col name for a15
SYS@ora12c> select con_id, name,open_mode from v$containers;
    CON_ID NAME            OPEN_MODE
---------- --------------- --------------------
         1 CDB$ROOT        READ WRITE
         2 PDB$SEED        READ ONLY
         3 SALESPDB        READ ONLY
         4 HRPDB           MOUNTED
```
<code>NO DATA</code>表示不从source PDB克隆数据，只Clone源数据。这个参数从12.1.0.2开始支持。
#### 1.3 从远程PDB Clone
本例中，远程PDB名称为SALESPDB，通过DBLINK远程连接至SALESPDB进行Clone。
```
#首先创建DBLINK
create database link salespdb
connect to system identified by oracle
using 'linora:1522/salespdb';
#将远程PDB处于read only状态
SQL> alter pluggable database salespdb close;
SQL> alter pluggable database salespdb open read only;
#在目标端执行Clone动作
CREATE PLUGGABLE DATABASE hrpdb FROM salespdb@salespdb no data
FILE_NAME_CONVERT = ('/u02/oradata/ora12c/salespdb/', '/u02/oradata/ora12c/hrpdb/')
STORAGE unlimited;
```
#### 1.4 从远程non-CDB Clone
Oracle有三种方法可以将non-CDB转换成PDB，包括data pump、OGG和DBMS_PDB包，这里简要说明下DBMS_PDB包的使用。这种方法只适合DB版本在12C以上。因为12C以下没有DBMS_PDB包。
```
#首先将non-CDB切换成read only状态
SYS> startup mount;
SYS> alter database open read only;
#生成XML
SQL> BEGIN
DBMS_PDB.DESCRIBE(pdb_descr_file => '/home/oracle/ncdb.xml');
END;
/
#检测生成的XML文件是否支持插拔
SQL> SET SERVEROUTPUT ON
DECLARE
hold_var boolean;
begin
hold_var := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(pdb_descr_file=>'/home/oracle/ncdb.xml');
if hold_var then
dbms_output.put_line('YES');
else
dbms_output.put_line('NO');
end if;
end;
/
#在CDB中创建PDB
SQL> CREATE PLUGGABLE DATABASE dkpdb
USING '/home/oracle/ncdb.xml'
COPY
FILE_NAME_CONVERT = ('/u01/dbfile/dk/',
'/u01/dbfile/CDB/dkpdb/');
#最后连接到新建的PDB，执行后续脚本
$ sqlplus sys/oralce@'linora:1522/dkpdb' as sysdba
SQL> @?/rdbms/admin/noncdb_to_pdb.sql
```
#### 1.5 插入已拔除的PDB
拔除指令：
```
SYS@ora12c> ALTER PLUGGABLE DATABASE hrpdb CLOSE IMMEDIATE;
Pluggable database altered.
SYS@ora12c> ALTER PLUGGABLE DATABASE hrpdb UNPLUG INTO '/home/ora12c/hrpdb.xml';
Pluggable database altered.
```
当某个PDB被拔除后，会保留在Mount状态，在原来的CDB中除了可以通过RMAN进行备份外，就只能drop，其他操作都会出现异常。如果要将拔除的PDB插回原CDB，需要将该PDB DROP，然后再插入。
```
#检测COMPATIBILITY
SQL> SET SERVEROUTPUT ON
DECLARE
hold_var boolean;
begin
hold_var := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(pdb_descr_file=>'/home/ora12c/hrpdb.xml');
if hold_var then
dbms_output.put_line('YES');
else
dbms_output.put_line('NO');
end if;
end;
/
YES
SYS@ora12c> drop pluggable database hrpdb;
Pluggable database dropped.
#在新CDB中插入拔除的CDB
CREATE PLUGGABLE DATABASE pdb2 using '/home/ora12c/hrpdb.xml'
move|copy
file_name_convert=('/u02/oradata/ora12c/hrpdb/','/u02/oradata/ora12c/pdb2/');
--source_file_name_convert=('/u02/oradata/ora12c/hrpdb/','/u02/oradata/ora12c/pdb2/') nocopy;
```
其中，move表示数据文件从原有位置mv至新位置，COPY则表示复制，NOCOPY表示重新生成。对于plug-in PDB有如下限制：
<li>源端和目标端CDB endianness必须一样</li>
<li>源端和目标端数据库选件必须一样</li>
<li>目标端必须存在原来PDB的数据文件</li>
<li>两端字符集和compatible 必须一样</li>
### 2. 连接PDB
PDB创建后会自动创建和PDB名称一致的Service，在TNSNAMES中添加此Service，即可通过tns方式连接：
```
[ora12c@linora:/u01/app/ora12c/product/12.1.0/db1/network/admin]$ cat listener.ora 
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = linora)(PORT = 1522))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1522))
    )
  )
ADR_BASE_LISTENER = /u01/app/ora12c

SID_LIST_LISTENER =
(SID_LIST = 
   (SID_DESC =
     (GLOBAL_DBNAME = ora12c)
     (ORACLE_HOME = /u01/app/ora12c/product/12.1.0/db1)
     (SID_NAME = ora12c)
     )
   (SID_DESC =
     (GLOBAL_DBNAME = pdb12c) #PDB service, the same as PDB database name
     (ORACLE_HOME = /u01/app/ora12c/product/12.1.0/db1)
     (SID_NAME = ora12c)   #CDB sid or the cdb database name
     )
)

salespdb =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.188)(PORT = 1522))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = salespdb)
    )
  )
[ora12c@linora:/home/ora12c]$ sqlplus sys/oracle@salespdb as sysdba

SYS@ora12c> col name for a20
SYS@ora12c> col NETWORK_NAME for a10
SYS@ora12c> col pdb for a20
SYS@ora12c> select name,NETWORK_NAME,PDB from cdb_services;
NAME                 NETWORK_NA PDB
-------------------- ---------- --------------------
SYS$BACKGROUND                  CDB$ROOT
SYS$USERS                       CDB$ROOT
ora12c               ora12c     CDB$ROOT
hrpdb                hrpdb      HRPDB
salespdb             salespdb   SALESPDB
```
或者使用简易连接方式，但是在sqlnet.ora需添加<code>ezconnect</code>：
```
[ora12c@linora:/home/ora12c]$ sqlplus salesadm/oracle@linora:1522/salespdb
SQL*Plus: Release 12.1.0.2.0 Production on Wed Aug 13 17:19:31 2014
Copyright (c) 1982, 2014, Oracle.  All rights reserved.
Last Successful login time: Wed Aug 13 2014 17:19:22 +08:00
Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options
SALESADM@linora:1522/salespdb>
```
或者使用以前的连接方式，通过alter session命令登入PDB：
```
SYS@ora12c> alter session set container=salespdb;
Session altered
SYS@ora12c> col CON_ID for a15
SYS@ora12c> col CUR_CONTAINER for a20
SYS@ora12c> col CUR_USER for a10
SYS@ora12c> SELECT SYS_CONTEXT('USERENV', 'CON_ID') AS con_id,
  2  SYS_CONTEXT('USERENV', 'CON_NAME') AS cur_container,
  3  SYS_CONTEXT('USERENV', 'SESSION_USER') AS cur_user
  4  FROM DUAL;
CON_ID          CUR_CONTAINER        CUR_USER
--------------- -------------------- ----------
3               SALESPDB             SYS
```
### 3. Start or shutdown PDB

```
#从root容器执行
SYS@ora12c> SELECT SYS_CONTEXT ('USERENV', 'CON_NAME') FROM DUAL;
SYS_CONTEX
----------
CDB$ROOT
SQL> alter pluggable database salespdb open;
SQL> startup pluggable database salespdb open read only;
SQL> alter pluggable database salespdb close immediate;
SQL> alter pluggable database all open;
SQL> alter pluggable database all close immediate;
#从PDB执行
SYS@ora12c> alter session set container=salespdb;
Session altered.
SYS@ora12c> SELECT SYS_CONTEXT ('USERENV', 'CON_NAME') FROM DUAL;
SYS_CONTEX
----------
SALESPDB
SQL> startup;
SQL> shutdown immediate;
```











Reference:  
[Creating and Removing PDBs with SQL*Plus](http://docs.oracle.com/database/121/ADMIN/cdb_plug.htm#ADMIN13561)
