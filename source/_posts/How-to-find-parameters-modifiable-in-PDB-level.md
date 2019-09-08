---
title: 12c PDB级别可修改参数
categories: oracle
comments: false
date: 2019-09-07 09:13:47
tags: multitenant
---
12C中，PDB级别可修改的参数有上百个，通过v$system_parameter可查询，但PDB级别没有单独SPFILE的。
当前数据库含有1个PDB：

```sql
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
    3 SDPDB1			  READ WRITE NO
```

尝试修改PDB级参数：

```sql
SQL> alter session set container = sdpdb1;
SQL> alter system set open_cursors=5000;
```
查看修改后的状态：
```sql
SQL> show parameter open_cursor

NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
open_cursors			     integer		    5000
SQL> alter session set container = cdb$root;

Session altered.

SQL> show parameter open_cursor

NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
open_cursors			     integer		    300
```
修改完后的PDB参数实际上存储在`pdb_spfile$`中：

```sql
col db_uniq_name for a10
col pdb_name for a10
col params for a20
col "VALUE$" for a20
set line 200 pagesize 200
select db_uniq_name,b.name pdb_name, sid, a.name params, value$
from pdb_spfile$ a, gv$pdbs b
where a.pdb_uid=b.con_uid;
```
一般建议使用v$parameter或者v$system_parameter去查看。
```sql
select name, con_id, value from v$system_parameter where name='open_cursors';
```

通过v$parameter或者v$system_parameter中的`ispdb_modifiable`栏位确定该参数是否在PDB级别修改。
```sql
select name, ispdb_modifiable
from v$parameter
where ispdb_modifiable='TRUE'
```


__EOF__
