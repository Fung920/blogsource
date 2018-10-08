---
layout: post
title: "RMAN完全恢复和不完全恢复"
date: 2014-06-06 17:20:28
comments: false
categories: oracle
tags: rman
keywords: rman recovery,RMAN恢复
description: demonstrate rman complete and incomplete recovery,RMAN完全恢复和不完全恢复
---
完全恢复和不完全恢复在前文[RMAN Recovery Concepts](/rman-recovery-concepts.html)已经简单描述过。本文简单描述下基于RMAN的完全恢复和不完全恢复。
<!--more-->
完全恢复一般用于物理介质故障，如介质失败，磁盘或者数据文件故障；不完全恢复一般用于逻辑业务故障，如用户误删除关键业务数据，或者用于日志不全的情况下的恢复，比如丢失数据文件，要做完全恢复，发现部分归档已经无法使用。此时只能恢复到可用归档日志的那一刻时间点。通过RMAN完全恢复，可以将数据库恢复到失败点状态；通过RMAN不完全恢复，可以将数据库恢复到备份点与失败点之间某个时刻的状态。下表列出了在数据库遇到故障时候应该采取何种恢复策略。
<center>Table 1-1 RMAN恢复策略</center>

需要介质恢复的文件|恢复动作
:---------------|:---------------:
数据文件，所有日志都可用|<center>完全恢复</center>
数据文件，部分日志不可用|<center>不完全恢复</center>
控制文件|<center>restore控制文件</center>
在线日志文件|<center>清除或者重建日志文件，或者不完全恢复，参照[Handling Online Redo Log Failures](/handling-online-redo-log-failures.html)</center>
SPFILE|<center>restore spfile</center>
归档日志|<center>从备份还原归档</center>
没有RMAN信息的控制文件(重建控制文件后)|<center>使用<code>catalog</code>命令或者<code>DBMS_BACKUP_RESTORE</code></center>
### 1.完全恢复
#### 1.1OPEN状态丢失数据文件
```
ORA-01116: error in opening database file 5
ORA-01110: data file 5: '/oradata/datafile/linora/fung01.dbf'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
SQL> select file#, status, error,recover from v$datafile_header;

     FILE# STATUS  ERROR                REC
---------- ------- -------------------- ---
         1 ONLINE                       NO
         2 ONLINE                       NO
         3 ONLINE                       NO
         4 ONLINE                       NO
         5 ONLINE  CANNOT OPEN FILE
         6 ONLINE                       NO
         7 ONLINE                       NO
```
在执行恢复前可先用preview命令查看需要那些备份集和归档。
```
RMAN> restore datafile 7 preview;
```
对单个数据文件执行恢复：
```
RMAN> sql 'alter database datafile 5 offline';
#如果原来磁盘路径不可用,则
#RMAN> set newname for datafile 5 to '/disk1/users01.dbf';
RMAN> restore datafile 5;
RMAN> recover datafile 5;
RMAN> sql 'alter database datafile 5 online';
SQL> select file#, status, error,recover from v$datafile_header;

     FILE# STATUS  ERROR                REC
---------- ------- -------------------- ---
         1 ONLINE                       NO
         2 ONLINE                       NO
         3 ONLINE                       NO
         4 ONLINE                       NO
         5 ONLINE                       NO
         6 ONLINE                       NO
         7 ONLINE                       NO

7 rows selected.
```
如果丢失的是1号数据文件，则需要在mount状态下进行恢复和还原。而对于数据文件及表空间在OPEN状态下进行完整恢复首先要OFFLINE对应的objects。
```
RMAN> run {
2> startup force mount;
3> restore datafile 1;
4> recover datafile 1;
5> sql 'alter database open';
6> }
```
#### 1.2整库的完整恢复
```
RMAN> startup mount;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open;
```
如果使用的是备份的控制文件而不是当前的控制文件，则需要resetlogs动作：
```
RMAN> startup nomount;
RMAN> restore controlfile from autobackup;
RMAN> alter database mount;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open resetlogs;
```
RMAN <code>restore</code>默认会从最新的备份还原文件，如果需要从旧的备份中恢复，可执行<code>until</code>子句，但这样会增加完全恢复的时间，一般是最新的备份出问题才会使用以前的备份进行完全恢复。针对还原恢复过程中产生的归档删除，可用以下命令进行删除:
```
RMAN> recover database delete archivelog;
#保留归档区最大500M空间，超过此大小，归档会被删除
RMAN> recover database delete archivelog maxsize 500m;
```
#### 1.3还原SPFILE及归档
#####还原SPFILE
```
#从autobackup恢复SPFILE到指定路径
RMAN> restore spfile to '/home/oracle/spfile.ora' from autobackup ;
#从RMAN备份中恢复
RMAN> list backup of spfile;

List of Backup Sets
===================

BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
116     Incr 0  9.61M      DISK        00:00:03     2014-06-01 10:20:32
        BP Key: 116   Status: AVAILABLE  Compressed: NO  Tag: TAG20140601T101934
        Piece Name: /oradata/backup/lv0_LINORA_20140601_123
  SPFILE Included: Modification time: 2014-06-01 10:18:52
  SPFILE db_unique_name: LINORA
RMAN> restore spfile to '/home/oracle/spfile.ora' from '/oradata/backup/lv0_LINORA_20140601_123' ;
```
#####还原归档日志
```
#还原所有RMAN已经备份的归档
RMAN> restore archivelog all;
#从sequence 50开始还原
RMAN> restore archivelog from sequence 50;
#指定sequence区间还原
RMAN> restore archivelog from sequence 5170 until sequence 5178 thread 1;
RMAN> restore archivelog sequence between 5170 and 5178 thread 1;
#如果需要强行覆盖现有的归档，则使用force子句
RMAN> restore archivelog from sequence 1 force;
#还原归档至非默认路径
RMAN> run{
2> set archivelog destination to '/ora01/archrest';
3> restore archivelog from sequence 5200;
4> }
```
### 2.不完全恢复
RMAN使用<code>until</code>子句，可基于时间，SCN，日志sequence和还原点进行不完全恢复。在restore中使用until子句，RMAN会根据until的时间点寻找合适的备份集进行恢复，如果restore不指定until，RMAN默认会从最新的可用备份还原。对于任何until子句，都只能恢复到这个点之前，而不包含这个点。  
使用不完全恢复，一定要数据库在mount状态进行。因为RMAN需要在mount状态下读写控制文件，同时，如果进行不完全恢复，system数据文件一般都要进行recovery，而system数据文件的recovery必须要OFFLINE。
#### 2.1基于时间的不完全恢复
```
run{
allocate channel c1 type disk;
set until time "to_date('2014-06-08 12:00:00','yyyy-mm-dd hh24:mi:ss')";
restore database;
recover database;
sql 'alter database open resetlogs';
release channel c1;
}
```
<code>set</code>子句一定要在run块里面。
#### 2.2基于日志sequence的恢复
```
run{
allocate channel c1 type disk;
set until sequence 65 thread 1;
restore database;
recover database;
sql 'alter database open resetlogs';
release channel c1;
}
```
#### 2.3基于scn值恢复
```
run{
allocate channel c1 type disk;
set until scn 1817351;
restore database;
recover database;
sql 'alter database open resetlogs';
release channel c1;
}
```
SCN和log sequence关系可根据如下两个视图查找：
```
select sequence#, first_change#, first_time
from v$log_history
order by first_time;

select sequence#, first_change#, first_time
from v$archived_log
order by first_time;
```
#### 2.4跨incarnation恢复
当数据库以resetlogs open的时候，会充值log sequence，在RMAN中，incarnation也就改变了。
```
RMAN> list incarnation;
List of Database Incarnations
DB Key  Inc Key DB Name  DB ID            STATUS  Reset SCN  Reset Time
------- ------- -------- ---------------- --- ---------- ----------
1       1       LINORA   3385851293       PARENT  1          2014-02-08 11:47:41
2       2       LINORA   3385851293       PARENT  869112     2014-03-07 10:35:25
3       3       LINORA   3385851293       PARENT  870364     2014-03-07 10:58:41
5       5       LINORA   3385851293       PARENT  870978     2014-03-07 11:35:43
4       4       LINORA   3385851293       ORPHAN  871371     2014-03-07 11:17:09
6       6       LINORA   3385851293       PARENT  1082046    2014-03-21 15:20:16
7       7       LINORA   3385851293       PARENT  1082598    2014-03-21 16:01:50
8       8       LINORA   3385851293       PARENT  1817350    2014-06-09 15:19:46
9       9       LINORA   3385851293       CURRENT 1817351    2014-06-09 15:26:20
```
当试图恢复到之前的incarnation时候，会报以下错误
```
RMAN> run{
2> allocate channel c1 type disk;
3> set until scn 1817350;
4> restore database;
5> recover database;
6> sql 'alter database open resetlogs';
7> release channel c1;
8> }

allocated channel: c1
channel c1: SID=10 device type=DISK

executing command: SET until clause

Starting restore at 2014-06-09 15:35:15
released channel: c1
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of restore command at 06/09/2014 15:35:15
RMAN-20208: UNTIL CHANGE is before RESETLOGS change
```
出现以上错误就表示我们需要从旧的incarnation进行恢复。首先要从对应incarnation中还原控制文件。
```
RMAN> restore controlfile from '/oradata/backup/lv0_LINORA_20140601_123' 
until time "to_date('2014-06-09 15:19:46','yyyy-mm-dd hh24:mi:ss')";

Starting restore at 2014-06-09 15:41:22
using channel ORA_DISK_1

channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
output file name=/oradata/datafile/linora/control01.ctl
output file name=/oradata/datafile/linora/control02.ctl
Finished restore at 2014-06-09 15:41:25
RMAN> sql 'alter database mount';
RMAN> list incarnation; 
List of Database Incarnations
DB Key  Inc Key DB Name  DB ID            STATUS  Reset SCN  Reset Time
------- ------- -------- ---------------- --- ---------- ----------
1       1       LINORA   3385851293       PARENT  1          2014-02-08 11:47:41
2       2       LINORA   3385851293       PARENT  869112     2014-03-07 10:35:25
3       3       LINORA   3385851293       PARENT  870364     2014-03-07 10:58:41
5       5       LINORA   3385851293       PARENT  870978     2014-03-07 11:35:43
4       4       LINORA   3385851293       ORPHAN  871371     2014-03-07 11:17:09
6       6       LINORA   3385851293       PARENT  1082046    2014-03-21 15:20:16
7       7       LINORA   3385851293       CURRENT 1082598    2014-03-21 16:01:50
```
在此例中，由于我们要恢复的时间点在2014-06-09，因此包含在incarnation 7中，不需要进行reset incarnation。如果需要恢复到2014-02-08，则需执行reset incarnation动作。
```
RMAN> reset database to incarnation 1;
```
此例中，直接进行不完全恢复即可
```
run{
allocate channel c1 type disk;
set until time "to_date('2014-06-09 15:19:46','yyyy-mm-dd hh24:mi:ss')";
restore database;
recover database;
sql 'alter database open resetlogs';
release channel c1;
}
RMAN> list incarnation; 
List of Database Incarnations
DB Key  Inc Key DB Name  DB ID            STATUS  Reset SCN  Reset Time
------- ------- -------- ---------------- --- ---------- ----------
1       1       LINORA   3385851293       PARENT  1          2014-02-08 11:47:41
2       2       LINORA   3385851293       PARENT  869112     2014-03-07 10:35:25
3       3       LINORA   3385851293       PARENT  870364     2014-03-07 10:58:41
5       5       LINORA   3385851293       PARENT  870978     2014-03-07 11:35:43
4       4       LINORA   3385851293       ORPHAN  871371     2014-03-07 11:17:09
6       6       LINORA   3385851293       PARENT  1082046    2014-03-21 15:20:16
7       7       LINORA   3385851293       PARENT  1082598    2014-03-21 16:01:50
8       8       LINORA   3385851293       CURRENT 1817350    2014-06-09 15:49:49
```
### 3.DBMS_BACKUP_RESTORE简单使用
如果没有使用recovery catalog数据库，在re-create控制文件后，控制文件的备份信息都会丢失，在10g以上版本中，可用<code>catalog start with</code>命令在RMAN注册备份信息。如果是9i以下，则没有catalog命令去同步备份信息，如果9i中重建控制文件后需要恢复，则需要使用到<code>DBMS_BACKUP_RESTORE</code>从指定备份片还原数据库所需文件到指定位置。
```
#For 10g and above
RMAN> catalog start with '/oradata/backup';

searching for all files that match the pattern /oradata/backup

List of Files Unknown to the Database
=====================================
File Name: /oradata/backup/lv0_LINORA_20140601_123
File Name: /oradata/backup/arch.sh
File Name: /oradata/backup/lv0_LINORA_20140601_124
File Name: /oradata/backup/full.sh

Do you really want to catalog the above files (enter YES or NO)? yes
```
#### 3.1还原控制文件
```
SQL> SET SERVEROUTPUT ON
SQL> DECLARE
  2  finished BOOLEAN;
  3  v_dev_name VARCHAR2(75);
  4  BEGIN
  5  -- Allocate a channel, when disk then type = null, if tape then type = sbt_tape.
  6  v_dev_name := dbms_backup_restore.deviceAllocate(type=>null, ident=>'d1');
  7  --
  8  dbms_backup_restore.restoreSetDatafile;
  9  dbms_backup_restore.restoreControlFileTo(
 10  cfname=>'/oradata/datafile/linora/control01.ctl');
 11  --
 12  dbms_backup_restore.restoreBackupPiece(
 13  '/oradata/backup/lv0_LINORA_20140601_123', finished);
 14  --
 15  if finished then
 16  dbms_output.put_line('Control file restored.');
 17  else
 18  dbms_output.put_line('Problem');
 19  end if;
 20  --
 21  dbms_backup_restore.deviceDeallocate('d1');
 22  END;
 23  /
Control file restored.

PL/SQL procedure successfully completed.
```
#### 3.2从单一备份片还原数据文件
```
SQL> SET SERVEROUTPUT ON
SQL> DECLARE
  2  finished BOOLEAN;
  3  v_dev_name VARCHAR2(75);
  4  BEGIN
  5  -- Allocate channels, when disk then type = null, if tape then type = sbt_tape.
  6  v_dev_name := dbms_backup_restore.deviceAllocate(type=>null, ident=> 'd1');
  7  --
  8  -- Set beginning of restore operation (does not restore anything yet).
  9  dbms_backup_restore.restoreSetDatafile;
 10  --
 11  -- Define datafiles and their locations for datafiles in first backup piece.
 12  dbms_backup_restore.restoreDatafileTo(dfnumber=>1,toname=>'/oradata/datafile/linora/system01.dbf');
 13  dbms_backup_restore.restoreDatafileTo(dfnumber=>2,toname=>'/oradata/datafile/linora/sysaux01.dbf');
 14  dbms_backup_restore.restoreDatafileTo(dfnumber=>4,toname=>'/oradata/datafile/linora/users01.dbf');
 15  --
 16  -- Restore the datafiles in this backup piece.
 17  dbms_backup_restore.restoreBackupPiece(done => finished,
 18  handle=>'/oradata/backup/lv0_LINORA_20140601_122', params=>null);
 19  --
 20  IF finished THEN
 21  dbms_output.put_line('Datafiles restored');
 22  ELSE
 23  dbms_output.put_line('Problem');
 24  END IF;
 25  --
 26  dbms_backup_restore.deviceDeallocate('d1');
 27  END;
 28  /
Datafiles restored

PL/SQL procedure successfully completed.
```
### 3.3从多个备份片还原数据文件
```
SET SERVEROUTPUT ON
DECLARE
finished BOOLEAN;
v_dev_name VARCHAR2(75);
TYPE v_filestable IS TABLE OF varchar2(500) INDEX BY BINARY_INTEGER;
v_filename V_FILESTABLE;
v_num_pieces NUMBER;
BEGIN
-- Allocate channels, when disk then type = null, if tape then type = sbt_tape.
v_dev_name := dbms_backup_restore.deviceAllocate(type=>null, ident=> 'd1');
--
-- Set beginning of restore operation (does not restore anything yet).
dbms_backup_restore.restoreSetDatafile;
--
-- Define backup pieces in backup set.
v_filename(1) :=
'/oradata/backup/lv0_LINORA_20140601_122';
v_filename(2) :=
'/oradata/backup/lv0_LINORA_20140601_123';
v_filename(3) :=
'/oradata/backup/lv0_LINORA_20140601_124';
-- There are 3 backup pieces in this backup set.
v_num_pieces := 3;
-- Define datafiles and locations.
dbms_backup_restore.restoreDatafileTo(dfnumber=>1,toname=>'/oradata/datafile/linora/system01.dbf');
dbms_backup_restore.restoreDatafileTo(dfnumber=>4,toname=>'/oradata/datafile/linora/users01.dbf');
-- Restore the datafiles in this backup set.
FOR i IN 1..v_num_pieces LOOP
dbms_backup_restore.restoreBackupPiece(done => finished, handle=> v_filename(i),
params=>null);
END LOOP;
--
IF finished THEN
dbms_output.put_line('Datafiles restored');
ELSE
dbms_output.put_line('Problem');
END IF;
--
dbms_backup_restore.deviceDeallocate('d1');
END;
/
```
### 3.4还原归档日志
```
SQL> SET SERVEROUTPUT ON
SQL> DECLARE
  2  finished BOOLEAN;
  3  v_dev_name VARCHAR2(75);
  4  BEGIN
  5  -- Allocate channels, when disk then type = null, if tape then type = sbt_tape.
  6  v_dev_name := dbms_backup_restore.deviceAllocate(type=>null, ident=> 'd1');
  7  --
  8  -- Set beginning of restore operation (does not restore anything yet).
  9  dbms_backup_restore.restoreSetArchivedlog;
 10  --
 11  -- Define archived redo log files to be restored.
 12  dbms_backup_restore.restoreArchivedlog(thread=>1, sequence=> 45);
 13  dbms_backup_restore.restoreArchivedlog(thread=>1, sequence=> 46);
 14  --
 15  dbms_backup_restore.restoreBackupPiece(done=>finished, handle=>
 16  '/oradata/backup/lv0_LINORA_20140601_119',
 17  params=>null);
 18  --
 19  IF finished THEN
 20  dbms_output.put_line('Archived redo log files restored');
 21  ELSE
 22  dbms_output.put_line('Problem');
 23  END IF;
 24  --
 25  dbms_backup_restore.deviceDeallocate('d1');
 26  END;
 27  /
Archived redo log files restored

PL/SQL procedure successfully completed.

SQL> !
[oracle@linora:/home/oracle]$ cd /oradata/arch/
[oracle@linora:/oradata/arch]$ ll
total 2048
-rw-r----- 1 oracle oinstall 2090496 Jun  9 16:22 1_45_842803310.arc
-rw-r----- 1 oracle oinstall    1024 Jun  9 16:22 1_46_842803310.arc
```
经过以上的restore，接下来就可以执行正常的recovery动作了，这个包在10g以上已经不建议使用，在9i的异机恢复中还是可以用上的。  
最后，备份很重要，有一个健全的备份机制策略，能最大限度的减少数据丢失的概率及数据库宕机的时间。  
Reference:  
[Oracle® Database Backup and Recovery User's Guide 11g Release 2 (11.2)](http://docs.oracle.com/cd/E11882_01/backup.112/e10642/toc.htm)
