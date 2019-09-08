---
title: Migrating Oracle ASM storage
categories: oracle
comments: false
date: 2019-07-31 08:44:10
tags: how to
---
This post is about how to migrate Oracle asm data to another storage, for my environment, is migrating from HHD storage to SSD storage, the redo blocksize also need to be modified to 4K for good performance.

<!--more-->

# 1. Preparation for migration
# 1.1 Backup whole database

```sql
RMAN>
backup database format '/backup/rman_full_%U' tag='db_full_bak'
including archive log all;
```

# 1.2 Verify file types and numbers in the ASM

```sql
-- grid user
sqlplus "/as sysasm"
select d.name group_name,f.type,count(*) cnt from v$asm_file f
    join v$asm_diskgroup d on f.group_number=d.group_number
    group by rollup (d.name,f.type);

--output example
GROUP_NAME                  TYPE                              CNT
--------------------------- -------------------------- ----------
CRS                         OCRFILE                             1
CRS                         DATAFILE                           11
CRS                         PASSWORD                            1
CRS                         TEMPFILE                            3
CRS                         ONLINELOG                           3
CRS                         CONTROLFILE                         1
CRS                         PARAMETERFILE                       1
CRS                         ASMPARAMETERFILE                    1
CRS                                                            22
FRA                         ONLINELOG                           8
FRA                         ARCHIVELOG                          9
FRA                         CONTROLFILE                         1
FRA                                                            18
DBDATA                      DATAFILE                           25
DBDATA                      PASSWORD                            1
DBDATA                      TEMPFILE                            6
DBDATA                      PARAMETERFILE                       1
DBDATA                                                         33
                                                               73
```

# 1.3 Verify asm_diskstrings parameter
Must include new storage path in the asm_diskstrings.

```sql
--grid user
show parameter asm_diskstring
```

# 1.4 Verify current ASM disks and diskgroups mapping information

```sql
--grid user
sqlplus "/as sysasm"
set line 200 pagesize 200
col name for a10
col path for a40
select group_number,name,OS_MB ,TOTAL_MB,FREE_MB ,HOT_USED_MB,COLD_USED_MB,PATH ,SECTOR_SIZE from v$asm_disk order by 1;

col PATH for a30
col DG_NAME for a10
col DG_STATE for a10
col FAILGROUP for a15
col name for a20
set lines 200 pages 10000

select dg.name dg_name, dg.state dg_state, dg.type, d.disk_number dsk_no,
d.path, d.mount_status, d.FAILGROUP,d.name, d.state
from v$asm_diskgroup dg, v$asm_disk d
where dg.group_number=d.group_number
order by dg_name, dsk_no;
```

## 1.5 Backup ASM diskgroup metadata and OCR/Voting disk information

* Query OCR and voting disk information

```sql
--root user
/u01/app/12.1.0/grid/bin/crsctl query css votedisk
/u01/app/12.1.0/grid/bin/ocrcheck
cat /etc/oracle/ocr.loc
```

* Backup OLR and OCR

Because voting disk is backed up by Oracle automatically, no need to backup again.
```sql
--root user
--check the backup
/u01/app/12.1.0/grid/bin/ocrconfig -showbackup
--backup olr
/u01/app/12.1.0/grid/bin/ocrconfig -local -manualbackup
--backup ocr
/u01/app/12.1.0/grid/bin/ocrconfig -manualbackup
```

* Backup asm diskgroup metadata

```sql
ASMCMD [+] > md_backup /home/grid/ocrvote.bak -G OCR
ASMCMD [+] > md_backup /home/grid/datadg.bak -G DBDATA
ASMCMD [+] > md_backup /home/grid/fradg.bak -G FRA
srvctl config asm > /home/grid/ocr_config.bak
```

* Backup ASM spfile

```sql
--grid user
$ORACLE_HOME/bin/gpnptool get -o- | xmllint --format - | grep -i spfile
asmcmd spget
sqlplus / as sysasm
SQL> create pfile = '/home/grid/pfile_asm.txt' from spfile;
```

# 2. Migrating ASM diskgroups

## 2.1 Adding new disk to diskgroups

```sql
--grid user
sqlplus / as sysasm
ALTER DISKGROUP fra ADD DISK   '/dev/mapper/fra_new_01', '/dev/mapper/fra_new_02';

--Monitoring rebalance progress
set line 3000
select * from gv$asm_operation;

--Speed up rebalance
alter diskgroup fra rebalance power 48;

--Put rebalance power back to 1 after rebalancing
alter diskgroup fra rebalance power 1;
```

## 2.2 Dropping old disk

```sql
--grid user
sqlplus / as sysasm

--It's better to drop disk one by one
ALTER DISKGROUP fra drop DISK 'FRA_0000';
ALTER DISKGROUP fra drop DISK 'FRA_0001';
```
Also, monitor and adjust rebalance power if needed.

For all the diskgroups in ASM, repeat step 2.1 - 2.2 one diskgroup by one diskgroup.

## 2.3 Verify migration result

```sql
sqlplus "/as sysasm"
set line 200 pagesize 200
col name for a10
col path for a40
select group_number,name,OS_MB ,TOTAL_MB,FREE_MB ,HOT_USED_MB,
COLD_USED_MB,PATH ,SECTOR_SIZE from v$asm_disk order by 1;

col PATH for a30
col DG_NAME for a10
col DG_STATE for a10
col FAILGROUP for a15
col name for a20
set lines 200 pages 10000

select dg.name dg_name, dg.state dg_state, dg.type, d.disk_number dsk_no,
d.path, d.mount_status, d.FAILGROUP,d.name, d.state
from v$asm_diskgroup dg, v$asm_disk d
where dg.group_number=d.group_number
order by dg_name, dsk_no;

select d.name group_name,f.type,count(*) cnt from v$asm_file f
    join v$asm_diskgroup d on f.group_number=d.group_number
    group by rollup (d.name,f.type);
```

# 3. Modify redo log file's blocksize

```sql
--oracle user
--query current log info
col member for a80
set line 200 pagesize 999
select a.inst_id,thread#,a.status, a.bytes/1024/1024,a.group#,b.type,member, a.blocksize
from gv$log a,gv$logfile b
where a.group#=b.group#
order by 5;
```

We need to add hidden parameter `_disk_sector_size_override` for modifing redo blocksize, this parameter is dynamic, no instance recycle needed.

```sql
alter system set "_disk_sector_size_override"=TRUE scope=both;
```

* Add new redo log

```sql
alter database add logfile thread 1 group 9 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 1 group 10 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 1 group 11 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 1 group 12 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 2 group 13 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 2 group 14 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 2 group 15 ('+FRA') size 2048M blocksize 4096;
alter database add logfile thread 2 group 16 ('+FRA') size 2048M blocksize 4096;
```

* Drop old redo groups

Ensure the log groups need to be dropped are in `inactive` status, by switching log, and checkpoint, we can put target redo groups in `inactive` state

```sql
SQL>
ALTER SYSTEM ARCHIVE LOG CURRENT;
ALTER SYSTEM ARCHIVE LOG CURRENT;
alter system checkpoint;
alter system checkpoint;

--Confirm target redo group is in inactive status
col member for a80
set line 200
select a.inst_id,thread#,a.status, a.bytes/1024/1024,a.group#,b.type,member, a.blocksize
from gv$log a,gv$logfile b
where a.group#=b.group#
order by 5;

--Drop old group
alter database drop logfile group 1;
alter database drop logfile group 2;
alter database drop logfile group 3;
alter database drop logfile group 4;
alter database drop logfile group 5;
alter database drop logfile group 6;
alter database drop logfile group 7;
alter database drop logfile group 8;

--Backup controlfile
alter database backup controlfile;
```

# 4. Risk assessment

* If ASM diskgroup is too large, rebalance maybe can't complete in the change window
* Setting rebalance power too high would cause bad performance

It's recommended to schedule multiple change window if data amount is too large.

# 5. Backout

Adding disks is a low risk operation, except rebalance power set to high and impact the performance. Under such circumstance, lower down rebalance power speed:

```sql
alter diskgroup DG_NAME rebalance power 1;
```

If something wrong with dropping disks, stop dropping operation by below command, only available for status = dropping in `v$asm_disk`:

```sql
ALTER DISKGROUP data1 UNDROP DISKS;
```

If something wrong with OCR or voting disk during migrating or RAC can't be started after migrating, follow below steps to restore OCR:

```sql
--root user, stop crs resource on all nodes
crsctl stop crs -f
crsctl stop resource ora.crsd -init

--root user, start crs in one node only in exclusive mode
crsctl start crs -excl -nocrs

--restore OCR
ocrconfig -restore OCR_BACKUP_FILENAME
--check the status
ocrcheck

--restore voting disk
crsctl replace votedisk +asm_disk_group
--check the status
crsctl query css votedisk

--restart RAC cluster
crsctl stop crs -f

crsctl stop cluster -all
crsctl start cluster -all

--check all nodes OCR with CVU tool
cluvfy comp ocr -n all -verbose
crsctl check cluster -all
```

# 6. Alternative way of migrating

Previous steps are available for online migration, we also can use rman copy to migrate, which need to stop database.

Assume we've create a new diskgroup named DATA_NEW, our purpose is to restore the whole database to DATA_NEW diskgroup.

* Backup database as copy into new diskgroup

```sql
run {
allocate channel c1 type disk;
allocate channel c2 type disk;
backup as copy database format '+data_new';
release channel c1;
release channel c2;
}

RMAN> list copy;
```

* After backing up, restart database to mount status

```sql
srvctl stop database -d DB_NAME
srvctl start database -d DB_NAME -startoptions mount
```

* Switch database to copy

```sql
RMAN>
switch database to copy;
RMAN>
recover database;
```

* Restart database in normal mode

```sql
srvctl stop database -d DB_NAME
srvctl start database -d DB_NAME
```



Reference:
[How To A Recreate Disk Group Used By CRS](https://blog.pythian.com/recreate-disk-group-crs/)
[OCR / Vote disk Maintenance Operations: (ADD/REMOVE/REPLACE/MOVE) (Doc ID 428681.1)](https://support.oracle.com/epmos/faces/SearchDocDisplay?_adf.ctrl-state=to8u2zc4y_4&_afrLoop=141918262651330#aref_section216)
[Software Patch Level and 12c Grid Infrastructure OCR Backup/Restore (Doc ID 1558920.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=142118453601784&parent=DOCUMENT&sourceId=428681.1&id=1558920.1&_afrWindowMode=0&_adf.ctrl-state=to8u2zc4y_144)
[Linux/Unix 平台，在CRS 磁盘组完全丢失后，如何恢复基于 ASM 的 OCR (Doc ID 2331776.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=to8u2zc4y_193&id=2331776.1&_afrLoop=142433806199044)

__EOF__
