---
title: 远程复制数据库到现有的CDB
#title: 远程复制数据库到CDB
categories: oracle
comments: false
date: 2019-08-13 17:22:38
tags: [how to, multitenant]
---
在12c中，针对PDB，Oracle提供了快捷便利的迁移方式。

以下分别示例了从传统数据库以及PDB下，迁移/复制数据库到目标CDB数据库。
所有的操作都是在standalone模式下完成，文中有关于RAC的仅限于理论。
数据库版本：12.1.0.2

<!--more-->

# 1. 限制
Oracle PDB的远程克隆有以下限制：

* 源库common user对应的表空间必须存在目标端中，否则复制过来的PDB会以受限模式开启(Bug 19174942)
* 如果源库是non-cdb，则源跟目标端版本必须在12.1.0.2以上
* 源库需处于read only模式; 如果是12.2，使用local undo, 启动归档，则可在正常模式下复制
* 目标端需建立到源库的DBLINK，这个dblink可以是到源库CDB或者PDB
* 对应的，目标端DBLINK的用户必须有create pluggable database的权限
* 两边需要有一致的字节顺序，字符集以及数据库的选件
     在测试过程中，连小补丁版本都需一致，否则，目标端需要重新升级/降级到与源库相同的版本

# 2. 复制远程non-cdb
复制前准备，查询两边数据库文件路径或者ASM磁盘名称，路径及名称不一致，需要添加`FILE_NAME_CONVERT`参数进行转换。
## 2.1 目标库创建DBLINK

```sql
CREATE DATABASE LINK orcl_lnk
CONNECT TO system IDENTIFIED BY manager USING 'orcl';
--remote user must be having create pluggable database privelege

-- test dblink
select name, open_mode from v$database@orcl_lnk;
```

## 2.2 将源库处于read only模式

* standalone

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE OPEN READ ONLY;
EXIT;
```

* RAC

```sql
srvctl stop database -d $DBNAME
srvctl start database -d $DBNAME -startoption "read only"
```

## 2.3 开始复制

```sql
CREATE PLUGGABLE DATABASE
tellerdb FROM orcl@orcl_lnk
--first orcl: name of non-cdb in source; second orcl_lnk: name for dblink in target
FILE_NAME_CONVERT= ('/oracle/oradata/orcl/tablespace','/oradata/orcldb/tellerdb','/oracle/oradata/orcl','/oradata/orcldb/tellerdb');
```

如果数据库文件是OMF命名，使用`create_file_dest`初始化参数去指定数据文件位置。
```sql
create pluggable database tellerdb from orcl@orcl_lnk
create_file_dest = '/oradata/orcldb/tellerdb';
```

## 2.4 开启源库到read write模式
复制完成后, 将源库重新启动到read write模式, 同时，检查日志是否有报错。

* standalone

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE OPEN ;
EXIT;
```

* RAC

```sql
srvctl stop database -d $DBNAME
srvctl start database -d $DBNAME
```

## 2.5 将non-cdb转换成pdb
由于源库是non-cdb,因此，需要执行以下脚本对其进行清理:

```sql
ALTER SESSION SET CONTAINER=tellerdb;
@$ORACLE_HOME/rdbms/admin/noncdb_to_pdb.sql
```

## 2.6 检查目标数据库PDB状态

```sql
alter pluggable database all open;
COLUMN name FORMAT A30
SELECT name, open_mode FROM v$pdbs WHERE name = 'TELLERDB';
```

检查补丁情况:

```sql
col time for a20
col name for a10
col message for a40
col action for a40
set line 200 pagesize 9999
select to_char(time) as time, name,message, status, action from pdb_plug_in_violations
where status <>'RESOLVED';
```

# 3. 远程复制PDB
## 3.1 目标库创建到PDB的dblink
```sql
CREATE PLUGGABLE DATABASE
sldb1 FROM ldb1@ldb1_zdb06
--ldb1: name for pdb in source db; ldb1_zdb06: name or dblink in target db
FILE_NAME_CONVERT= ('/oradata/zdb06','/oradata/hdkcdb');

--检查dblink
select name, open_mode from v$database@ldb1_zdb06;
```

## 3.2 源库pdb置于read only模式
```sql
alter pluggable database ldb1 close immediate instances=all;
alter pluggable database ldb1 open read only instances=all;
--alter pluggable database all except PDB1 close immediate instances=all;
--alter pluggable database all except PDB1 open read only instances=all;
```

## 3.3 复制pdb

```sql
CREATE PLUGGABLE DATABASE
ldb1 FROM ldb1@ldb1_zdb06
FILE_NAME_CONVERT= ('/oradata/zdb06','/oradata/hdkcdb');
```

## 3.4 重启源库到read write状态

克隆完成，将源数据库处于read write状态
```sql
alter pluggable database ldb1 close immediate instances=all;
alter pluggable database ldb1 open read write instances=all;
```

## 3.5 检查目标PDB状态

```sql
ALTER PLUGGABLE DATABASE ldb1 OPEN;
SELECT name, open_mode FROM v$pdbs WHERE name = 'LDB1';

col time for a20
col name for a10
col message for a40
col action for a40
set line 200 pagesize 9999
select to_char(time) as time, name,message, status, action from pdb_plug_in_violations
where status <>'RESOLVED';
```

# 4. 总结
* 1. NON-CDB跟PDB步骤都差不多，只不过non-cdb需要执行一个清理转换脚本。
* 2. 在复制过过程中，出现有补丁版本不一致导致还原后的PDB一直处于restricted模式
    从`pdb_plug_in_violations`，出现
    ```sql
    PSU bundle patch 170418 (DATABASE PATCHSET UPDATE 12.1.0.2.170418):
    Installed in the CDB but not in the PDB
    ```
    - 原因
        在我的环境里，non-cdb及pdb的源库均打了相关的补丁，但是呢，打补丁的同事最后并没有应用datapatch，因此，在数据字典中无法识别
    - 解决
        第一次复制non-cdb, 在目标库中，我尝试rollback之前的补丁，然后在打源库对应的补丁，问题能得到解决。
        第二次复制pdb, 我尝试在目标库中apply对应的补丁,在源库应用datapatch，之后再复制,复制过后的pdb状态正常。
    - 建议
        利用远程复制来创建PDB，都是基于物理复制，建议数据库小版本都要一致。

    如果需要单独对某个pdb进行datapatch的apply或者回滚，请参照以下命令：
    ```sql
    datapatch -rollback 28790654 -pdbs SLDB2 -force -verbose
    $ORACLE_HOME/OPatch/datapatch -apply 29251241 -pdbs SLDB2 -force -verbose
    ```

* 3. 复制过程中出现ora-01031

```sql
ERROR at line 2:
ORA-17628: Oracle error 1031 returned by remote Oracle server
ORA-01031: insufficient privileges
```

在源库给dblink用户赋予必须权限:
```sql
GRANT CREATE SESSION, SYSOPER, CREATE PLUGGABLE DATABASE TO system;
```

## 4.1 远程复制前的checklist
* 对比源跟目标库cdb/pdb下`dba_registry_sqlpatch`记录
* 对比源跟目标库DBBP/PSU版本
* 当存在OJVM补丁时候，考虑回滚目标端OJVM补丁，再进行复制
* DBLINK用户需有sysoper或则create pluggable database的权限



Reference:
[How to clone PDB ( Remote Clone ) across CDB using Database Link (Doc ID 2297470.1)](https://support.oracle.com/epmos/faces/SearchDocDisplay?_adf.ctrl-state=15i0wfigbx_9&_afrLoop=518696371637532#GOAL)
[How to Clone a Pluggable Database (PDB) Without Closing the PDB (Doc ID 1958865.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=520432551190083&parent=DOCUMENT&sourceId=2297470.1&id=1958865.1&_afrWindowMode=0&_adf.ctrl-state=15i0wfigbx_153)
[Example for Cloning PDB from NON-CDB via Dblink (Doc ID 1928653.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=520495376336185&parent=WIDGET_REFERENCES&sourceId=2297470.1&id=1928653.1&_afrWindowMode=0&_adf.ctrl-state=15i0wfigbx_202)

__EOF__

