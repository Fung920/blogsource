---
layout: post
title: "RMAN恢复基本概念"
date: 2014-06-01 12:34:25
comments: false
categories: oracle
tags: rman
keywords: rman,restore,recover,oracle,rman恢复
description: how to use rman to restore and recover database，rman恢复
---
大部分recovery场景都需要经过两个阶段：restore(还原)和recover(恢复)。
<!--more-->

|Condition On Start-up                          | Oracle Behavior                 |       DBA action|
|:-----------------------------------------------|---------------------------------|-----------------|
|CF checkpoint scn < datafile Checkpoint scn    |    Controlfile too old error    |      restore a newer controlfile|
|CF checkpoint scn >  datafile Checkpoint scn   |    Media recovery required      |      Most likely a datafile has been restored from a backup.Recovery is now required.|
|CF checkpoint scn = datafile checkpoint scn    |    Startup normally             |      None|
|Database mounted, instance thread status=OPEN  |    Crash recovery needed        |      NONE|


在RMAN中，还原和恢复具有不同的含义。还原是指访问之前生产的备份集，从中得到一个或者多个对象，然后在磁盘的某个位置还原这些对象，包括数据文件，控制文件，归档日志，SPFILE等。恢复是指重新应用事务的重做日志到数据文件的过程。
### 1. 完全恢复和不完全恢复
完全恢复意味着你能恢复所有在失败前已提交的事务。因此，完全恢复是不需要将所有数据文件都还原并且恢复的。我们需要做的只是恢复那些损坏了的数据文件。Oracle通过对比控制文件中的SCN值和dafatile header中的SCN值来确定哪些数据文件需要被恢复。
```sql
SQL> select file#,status,checkpoint_change#,
  2  to_char(checkpoint_time,'yyyy-mm-dd hh24:mi:ss') 
  3  from v$datafile_header;

     FILE# STATUS  CHECKPOINT_CHANGE# TO_CHAR(CHECKPOINT_
---------- ------- ------------------ -------------------
         1 ONLINE             1559658 2014-06-04 14:51:50
         2 ONLINE             1559658 2014-06-04 14:51:50
         3 ONLINE             1559658 2014-06-04 14:51:50
         4 ONLINE             1559658 2014-06-04 14:51:50
         5 ONLINE             1559658 2014-06-04 14:51:50
         6 ONLINE             1559658 2014-06-04 14:51:50
         7 ONLINE             1559658 2014-06-04 14:51:50
         8 ONLINE             1559658 2014-06-04 14:51:50
```
不完全恢复意味着不是所有已提交事务都能进行恢复，所以肯定丢失部分数据。不完全恢复一般用在指定时间点的恢复，由于误操作而导致的数据丢失，可以采用不完全恢复至数据删除前一刻。不完全恢复是通过指定recover database until命令进行的。所以不完全恢复也被称之为基于时间点的恢复(database point-in-time recovery)。  
在选择何种恢复类型前(有些时候没得选择)，首先要明白Oracle数据库的一些后台进程及它们的工作原理。因为RMAN的完全恢复和不完全恢复都是基于日志的，对日志的操作，必不可少需要对一些后台进程有进一步的了解。  
只要细心的用户都会发现，无论多大多长久的事务，commit都会很快完成。这是因为Oracle对缓存区的dirty block，在它们提交的时候，并不确保这些数据都已经写入磁盘了，而是保证这些事务所有的redo entry都已经被写入online log，这样，在实例失败或者介质失败的时候，它可以通过日志文件中的redo entry重演事务的变化。  
data buffer cache和redo buffer cache目的完全不同，data buffer cache是为了能将用户需要的数据块尽可能的保留长久，以提高性能，而redo buffer cache是为了缓存重做日志，以提高日志文件写速度。因此，DBWn是在Checkpoint发生时候才会写脏数据至磁盘，而LGWR则是频繁地将缓冲区的redo写入到磁盘。当Checkpoint发生时，ckpt后台进程记录当前控制文件及数据文件头SCN值。Checkpoinnt能保证数据在这一时间点是一致性状态。DBWn和redo的关系可参照前文[管理redo文件](/redo-manage.html)。  
### 2. Crash Recovery
<code>Crash Recovery</code>等同于单实例的<code>Instance Recovery</code>，很多人认为它们是同一概念，在很多情况下，这两者并没有区别，唯一例外的情况是在RAC环境中，<code>Instance Recovery</code>指的的某一节点crash，其他节点通过日志前滚和回滚失败节点的事务。而<code>Crash Recovery</code>是对单实例或者RAC所有节点crash后的恢复。<code>Crash Recovery</code>是通过Oracle的SMON进程自动完成的，用户不需要参与。当掉电或者数据库没有干净的关闭(shutdown abort)时，Oracle需要进行<code>Crash Recovery</code>，将数据恢复到失败前的一致性状态，即所有在失败前已经提交的数据写入磁盘，未提交的数据回滚。以下为一个实例恢复的例子：
```
SQL> shutdown abort
ORACLE instance shut down.
SQL> startup 
#alert.log
Beginning crash recovery of 1 threads
 parallel recovery started with 2 processes
Started redo scan
Completed redo scan
 read 36 KB redo, 31 data blocks need recovery
Started redo application at
 Thread 1: logseq 52, block 30273
Recovery of Online Redo Log: Thread 1 Group 1 Seq 52 Reading mem 0
  Mem# 0: /oradata/datafile/linora/redo01.log
Completed redo application of 0.01MB
Completed crash recovery at
 Thread 1: logseq 52, block 30346, scn 1584317
 31 data blocks read, 31 data blocks written, 36 redo k-bytes read
...
SMON: enabling cache recovery
...
SMON: enabling tx recovery
```
实例崩溃的恢复包含两个阶段：前滚(roll forward)和回滚(roll back)。<code>SMON</code>会自动将上一次Checkpoint以后的online redo log进行前滚和应用，这个过程就是通过redo信息重新把数据库加载到缓冲区，然后对已经提交的数据写入数据文件。前滚完成后，Oracle通过undo或者rollback segment对未提交的事务进行回滚。  
只要将数据文件头部的检查点SCN和当前的在线日志最新的重做记录的SCN进行对比，便可得知该数据文件是否为'旧'，如果是，则表示不同步，需要前滚。前滚包括以下步骤：
<li>读取数据文件头部检查点RBA，将RBA所指向的日志文件中的重做记录确定为前滚起点</li>
<li>根据RBA读取重做记录，获得改变向量中的AFN和DBA，得知哪些数据块需要被修改(重做)</li>
<li>根据改变向量中的DBA将数据块从数据文件中读至缓存，并比较数据块的实际版本(SCN+SEQ)与改变向量中记录的数据块版本</li>
<li>如版本相同，进入下一步骤。如改变向量的版本低于数据块的版本，Oracle会跳过当前重做记录，直接读取下一条重做记录；如果改变向量的版本高于数据块版本，前滚报错，crash recovery将意外终止</li>
<li>按顺序执行所有改变向量的操作，修改那些向量中DBA对应的数据块</li>
<li>重复上述步骤，直到用完在线日志的最后一条重做记录</li>
前滚完成后，已经提交的变更和没有提交的变更全部以数据块的形式从redo中恢复回来，接下来的任务就是回滚尚未提交的事务。关于RBA、DBA和重做记录及改变向量，可参照[重做记录](/redo-record.html)。  
在Oracle启动过程中，通过对比控制文件和datafile header的SCN值 [^1] 决定是正常开启数据库还是进行实例恢复或者进行介质恢复。两者的SCN值跟恢复动作关系如下表所示：  

<center>SCN Oracle Start-up Checks</center>

|Condition on Start-Up|Oracle Behavior|DBA Action|
|----|----|----|
|CF checkpoint SCN < Datafile checkpoint SCN|"Control file too old" error|Restore a newer control file.|
|CF checkpoint SCN > Datafile checkpoint SCN|Media recovery required|Most likely a datafile has been restored from a backup.Recovery is now required.|
|CF checkpoint SCN = Datafile SCN|Start up normally|None.|
|Database in mount mode, instance thread status = OPEN|Crash recovery required|None.  |

以下查询语句能找出当前数据库在mount状态下，是正常开启还是需要介质恢复。
```
SELECT
a.name,
a.checkpoint_change#,
b.checkpoint_change#,
CASE
WHEN ((a.checkpoint_change# - b.checkpoint_change#) = 0) THEN 'Startup Normal'
WHEN ((a.checkpoint_change# - b.checkpoint_change#) > 0) THEN 'Media Recovery'
WHEN ((a.checkpoint_change# - b.checkpoint_change#) < 0) THEN 'Old Control File'
ELSE 'what the ?'
END STATUS
FROM v$datafile a, -- control file SCN for datafile
v$datafile_header b -- datafile header SCN
WHERE a.file# = b.file#;
```
以下动态性能视图能帮忙找到需要介质恢复的数据文件
```
--from data file hearder
select file#, status, error,recover from v$datafile_header;
--from controlfile
select file#, error from v$recover_file;
```
在启动阶段，Oracle检查实例的日志thread状态来决定crash recovery是否需要。当数据库正常开启时候，thread的状态是OPEN，当正常关闭时候，thread状态是CLOSED。当数据库异常终止，如shutdown abort，thread状态仍旧保持OPEN，在启动阶段，SMON检测到thread状态不正常，需要进行实例崩溃的恢复。
```
#数据库OPEN状态
SQL> SELECT
  2  a.thread#, b.open_mode, a.status,
  3  CASE
  4  WHEN ((b.open_mode='MOUNTED') AND (a.status='OPEN')) THEN 'Crash Recovery req.'
  5  WHEN ((b.open_mode='MOUNTED') AND (a.status='CLOSED')) THEN 'No Crash Rec. req.'
  6  WHEN ((b.open_mode='READ WRITE') AND (a.status='OPEN')) THEN 'Inst. already open'
  7  ELSE 'huh?'
  8  END STATUS
  9  FROM v$thread a,
 10  v$database b,
 11  v$instance c
 12  WHERE a.thread# = c.thread#;

   THREAD# OPEN_MODE            STATUS STATUS
---------- -------------------- ------ -------------------
         1 READ WRITE           OPEN   Inst. already open
#数据库正常关闭
SQL> shutdown immediate
SQL> startup mount
SQL> SELECT
  2  a.thread#, b.open_mode, a.status,
  3  CASE
  4  WHEN ((b.open_mode='MOUNTED') AND (a.status='OPEN')) THEN 'Crash Recovery req.'
  5  WHEN ((b.open_mode='MOUNTED') AND (a.status='CLOSED')) THEN 'No Crash Rec. req.'
  6  WHEN ((b.open_mode='READ WRITE') AND (a.status='OPEN')) THEN 'Inst. already open'
  7  ELSE 'huh?'
  8  END STATUS
  9  FROM v$thread a,
 10  v$database b,
 11  v$instance c
 12  WHERE a.thread# = c.thread#;

   THREAD# OPEN_MODE            STATUS STATUS
---------- -------------------- ------ -------------------
         1 MOUNTED              CLOSED No Crash Rec. req.
#数据库异常关闭
SQL> shutdown abort
ORACLE instance shut down.
SQL> startup mount
SQL> SELECT
  2  a.thread#, b.open_mode, a.status,
  3  CASE
  4  WHEN ((b.open_mode='MOUNTED') AND (a.status='OPEN')) THEN 'Crash Recovery req.'
  5  WHEN ((b.open_mode='MOUNTED') AND (a.status='CLOSED')) THEN 'No Crash Rec. req.'
  6  WHEN ((b.open_mode='READ WRITE') AND (a.status='OPEN')) THEN 'Inst. already open'
  7  ELSE 'huh?'
  8  END STATUS
  9  FROM v$thread a,
 10  v$database b,
 11  v$instance c
 12  WHERE a.thread# = c.thread#;

   THREAD# OPEN_MODE            STATUS STATUS
---------- -------------------- ------ -------------------
         1 MOUNTED              OPEN   Crash Recovery req.
```
### 3. 确认备份集
在恢复还原前，确认哪些备份集和文件是需要在恢复过程中用到的。preview的命令类似RMAN的list命令。
```
RMAN> restore database preview;
RMAN> restore database preview summary;
RMAN> restore database preview;
RMAN> restore database from tag TAG20140601T101934 preview;
RMAN> restore datafile 1, 2, 3, 4 preview;
RMAN> restore archivelog all preview;
RMAN> restore archivelog from time 'sysdate - 1' preview;
RMAN> restore archivelog from scn 3243256 preview;
RMAN> restore archivelog from sequence 29 preview;
```
检查备份的完整性，仅仅读取备份文件，不会真正做restore。
```
RMAN> restore database validate;
RMAN> validate backupset 115;
RMAN> restore database from tag TAG20140601T101934 validate;
RMAN> restore datafile 1 validate;
RMAN> restore archivelog all validate;
RMAN> restore controlfile validate;
RMAN> restore tablespace users validate;
```
以上的检查只是物理性的检查，如果需要逻辑上检测备份集是否有问题，可用check logical子句，并且通过V$DATABASE_BLOCK_CORRUPTION查看检查结果。
```
restore database validate check logical;
```

To Be Continued!
[^1]: <code>V$DATAFILE_HEADER</code>以数据文件为参考源.<code>V$DATAFILE</code>以控制文件为参考源.需要两者的<code>checkpoint_change#</code>相同才能正常开启数据库.

