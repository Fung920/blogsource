---
title: 利用rman增量备份恢复adg gap
categories: oracle
comments: false
date: 2019-09-04 17:43:08
tags: ['rman','dataguard','how to']
---
由于网络变更，导致部分ADG链路中断，在部分数据库上，没有配置RMAN的删除归档策略，主库归档已经被删除，备库无法继续应用日志。

## 1. 错误描述
使用`dgmgrl`监控adg状态，出现以下错误：
```sql
DGMGRL> show configuration;
Configuration - ADG_CONFIG
  Protection Mode: MaxAvailability
  Databases:
    CDB01 - Primary database
      Error: ORA-16810: multiple errors or warnings detected for the database
    CDB02 - Physical standby database
Fast-Start Failover: DISABLED
Configuration Status:
ERROR

DGMGRL> show database 'CDB01';
Database - CDB01
  Role:            PRIMARY
  Intended State:  TRANSPORT-ON
  Instance(s):
      cdb011
      cdb012
  Database Error(s):
    ORA-16783: cannot resolve gap for database cdb02
  Database Warning(s):
    ORA-16629: database reports a different protection level from the protection mode
Database Status:
ERROR
```

通过以下SQL确实有gap：
```sql
set line 200 pagesize 200
col hostname for a30
with
    db as (SELECT NAME DB_NAME, database_role db_role FROM GV$DATABASE),
    inst as (SELECT inst_id,
            UPPER(SUBSTR(HOST_NAME, 1, (DECODE(INSTR(HOST_NAME, '.'), 0, LENGTH(HOST_NAME), (INSTR(HOST_NAME, '.') - 1))))) HOSTNAME
           FROM GV$INSTANCE),
    log1 as (SELECT thread#, MAX(SEQUENCE#) LOG_ARCHIVED
         FROM GV$ARCHIVED_LOG
         WHERE DEST_ID = 1
            AND ARCHIVED = 'YES'
         group by thread#),
    log2 as(
        SELECT thread#, MAX(SEQUENCE#) LOG_APPLIED
        FROM GV$ARCHIVED_LOG
        WHERE DEST_ID = 2
            AND APPLIED = 'YES'
        group by thread#),
    log3 as (SELECT thread#,TO_CHAR(MAX(COMPLETION_TIME), 'yyyy-mm-dd HH24:MI') APPLIED_TIME
        FROM GV$ARCHIVED_LOG
        WHERE DEST_ID = 2
            AND APPLIED = 'YES'
        group by thread#)
select distinct DB_NAME, db_role, HOSTNAME, LOG_ARCHIVED, LOG_APPLIED,APPLIED_TIME, LOG_ARCHIVED - LOG_APPLIED LOG_GAP
  from db, inst, log1, log2, log3
 where log1.thread# = log2.thread#
   and log2.thread# = log3.thread#
   and inst.inst_id = log1.thread#;

DB_NAME   DB_ROLE          HOSTNAME             LOG_ARCHIVED LOG_APPLIED APPLIED_TIME        LOG_GAP
--------- ---------------- -------------------- ------------ ----------- ---------------- ----------
CDB01     PRIMARY          SZDB1                     3209        3184 2019-09-01 03:28         25
CDB01     PRIMARY          SZDB2                     1862        1835 2019-09-01 11:22         27
```

## 2. 解决方案
### 2.1 停止standby应用日志
Standby操作：
```sql
dgmgrl /
edit database cdb02 set state='APPLY-OFF';
show database cdb02
```

### 2.2 主库从备库最后SCN进行增量备份
备库查找当前SCN值:
```sql
select current_scn from gv$database;
```

主库开始备份:
```sql
--primary
run{
allocate channel c1 type disk;
backup incremental from scn 5125186024 database format '/tmpbak/backup/standby_%d_%t_%c_%p';
}
```

### 2.3 备库进行recover
将备份片传送到备库相应位置，对备库进行还原:
* 重启备库到mount状态
```sql
srvctl stop database -d cdb02
srvctl start database -d cdb02 -startoption mount
```

* 对备库进行recover
```sql
--standby
rman target /
catalog start with '/home/oracle/backup';
run{
allocate channel c1 type disk;
allocate channel c2 type disk;
recover database noredo;
}
```

### 2.4 备库restore控制文件
```sql
--primary
--主库备份standby控制文件
alter database create standby controlfile as '/home/oracle/standby.ctl';

--重启备库，仅让其中一个节点启动到nomount状态
srvctl stop database -d cdb02
startup nomount
show parameter control_files

RMAN> restore controlfile from '/home/oracle/standby.ctl';
```

### 2.5 catalog备库数据文件
控制文件重新还原过来，需要对数据库文件进行catalog，因为数据文件路径不一致需要对数据文件进行重命名。
```sql
--standby
MAN> alter database mount;
RMAN> catalog start with '+DATA2/CDB02';
RMAN> switch database to copy;
```

由于路径不一致，有可能导致日志无法写入备库，建议在此时添加`log_file_name_convert`参数。
```sql
alter system set log_file_name_convert ='+FRA/CDB01','+FRA/CDB02' scope=spfile sid='*';
```
在dgbroker启用后，同时要调整broker参数：
```sql
edit database cdb02 set property 'LogFileNameConvert'='+FRA/CDB01,+FRA/CDB02';
```

重启数据库到RAC模式:
```sql
shutdown immediate;
srvctl start database -d cdb02
```

### 2.6 清空备库standby日志
```sql
--standby
alter database clear logfile group 10;
alter database clear logfile group 11;
alter database clear logfile group 12;
alter database clear logfile group 13;
alter database clear logfile group 14;
alter database clear logfile group 15;
alter database clear logfile group 16;
```

### 2.7 开启备库日志应用
```sql
edit database cdb02 set state='APPLY-ON';
```

## 3. Trouble shooting
### 3.1 备库日志路径与主库不一致
主库后台日志报以下错误：
```sql
Wed Sep 04 16:25:03 2019
Errors in file /u01/app/oracle/diag/rdbms/cdb01/cdb011/trace/cdb011_arc0_153792.trc:
ORA-16041: Remote File Server fatal error
Wed Sep 04 16:25:03 2019
ARC0: FAL archive failed with error 16041.  See trace for details
Wed Sep 04 16:25:03 2019
Errors in file /u01/app/oracle/diag/rdbms/cdb01/cdb011/trace/cdb011_arc0_153792.trc:
ORA-16055: FAL request rejected
ARCH: FAL archive failed. Archiver continuing
```
DGBroker报以下错误：
```sql
DGMGRL> show database agxxxprof

 Database - cdb02
 Role: PHYSICAL STANDBY
 Intended State: APPLY-ON
 Transport Lag: 0 seconds (computed 1 second ago)
 Apply Lag: 0 seconds (computed 1 second ago)
 Average Apply Rate: 2.12 MByte/s
 Real Time Query: OFF
 Instance(s):
      cdb02
         Database Warning(s): ORA-16789: standby redo logs configured incorrectly
 Database Status: WARNING
```

解决方法：
主库重新添加备库日志，如果主备库路径不一致，备库需要添加`log_file_name_convert`参数进行路径转换。
```sql
alter database drop standby logfile group 1;
alter database drop standby logfile group 2;
alter database drop standby logfile group 3;
alter database drop standby logfile group 4;
alter database drop standby logfile group 5;
alter database drop standby logfile group 6;
alter database drop standby logfile group 7;
alter database drop standby logfile group 8;

alter database add standby logfile thread 1 group 1 ('+FRA') size 4096M blocksize 4096;
alter database add standby logfile thread 1 group 2 ('+FRA') size 2048M blocksize 4096;
alter database add standby logfile thread 1 group 3 ('+FRA') size 2048M blocksize 4096;
alter database add standby logfile thread 1 group 4 ('+FRA') size 2048M blocksize 4096;
alter database add standby logfile thread 2 group 5 ('+FRA') size 2048M blocksize 4096;
alter database add standby logfile thread 2 group 6 ('+FRA') size 2048M blocksize 4096;
alter database add standby logfile thread 2 group 7 ('+FRA') size 2048M blocksize 4096;
alter database add standby logfile thread 2 group 8 ('+FRA') size 2048M blocksize 4096;
```

__EOF__
