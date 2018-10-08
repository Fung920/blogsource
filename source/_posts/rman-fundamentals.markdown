---
layout: post
title: "RMAN基础知识"
date: 2014-05-31 11:08:41
comments: false
categories: oracle
tags: rman
keywords: rman,backup,recover
description: oracle rman basic knowledges,RMAM基础知识,RMAN Fundamentals
---
### 1. Concepts
<!--more-->
##### Backup Sets & Image copies
RMAN backup的命令能备份出两种类型：备份集和镜像副本。RMAN默认会备份为备份集。备份集是一个逻辑结构，是由一个或者多个备份片组成，备份片则是真实存储备份数据的物理文件。如下所示，表示备份集47包含了1-6个数据文件，由一个备份片组成(/oradata/backup/bk_20140519_32_848006921_LINORA)：
```
RMAN> list backup of database;

using target database control file instead of recovery catalog

List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
47      Full    962.68M    DISK        00:01:13     2014-05-19 21:29:54
        BP Key: 47   Status: AVAILABLE  Compressed: NO  Tag: HOT_DB_BK_LEVEL0
        Piece Name: /oradata/backup/bk_20140519_32_848006921_LINORA
  List of Datafiles in backup set 47
  File LV Type Ckp SCN    Ckp Time            Name
  ---- -- ---- ---------- ------------------- ----
  1       Full 1282313    2014-05-19 21:28:41 /oradata/datafile/linora/system01.dbf
  2       Full 1282313    2014-05-19 21:28:41 /oradata/datafile/linora/sysaux01.dbf
  3       Full 1282313    2014-05-19 21:28:41 /oradata/datafile/linora/undotbs01.dbf
  4       Full 1282313    2014-05-19 21:28:41 /oradata/datafile/linora/users01.dbf
  5       Full 1282313    2014-05-19 21:28:41 /oradata/datafile/linora/fung01.dbf
  6       Full 1282313    2014-05-19 21:28:41 /oradata/datafile/linora/undotbs02.dbf
```
一个备份集能够包含一个或者多个数据文件，归档日志文件，控制文件。一个备份集默认只有一个备份片。同时，也可以通过<code>maxpiecesize</code>参数限制备份片的大小，超过这个参数值，一个备份片就会被分为多个备份片。指定<code>fileperset</code>参数则表示一个备份集中包含几个输入文件(datafile,archivelog等)。  
Backup set和Image copies最关键的区别在于，一个备份集可以同时从多个文件中的数据块写入到一个备份集中，而Image copy则只能是byte by byte，image copy就像OS的dd命令一样，属于完全一致的镜像复制。  
RMAN默认是以备份集的方式(backup as backupset)备份的，它只会备份分配了的数据块(处于高水位下的空块也包括)，而不是备份整个数据文件，相反，Image copy则是备份整个数据文件的所有块。由于在现实生产环境用Image copy用的少，在后续的讨论中，会忽略image copy的相关问题。
### 2. RMAN backup Modes
RMAN的备份模式按照数据一致性可分为一致性(consistent)备份和非一致性(inconsistent)备份，按照增量水平可分为全备和增量备份，增量备份又分为差异增量(Differential Incremental)备份和累积增量(Cumulative Incremental)备份。  
##### 一致性备份
当数据库干净的关闭后，然后进入mount模式进行的备份，这种备份模式如果在还原的时候是不需要进行recover的。  
##### 非一致性备份
当数据库处于open(需要在归档模式下)或者非干净的关闭后的mount下进行的备份，非一致性备份在restore完后都是需要进行recover动作的。  
因此，在RMAN的<code>backup</code>命令中，如果处于归档模式，则数据库在open或者mount状态下均可备份，如果处于非归档模式，则需要干净的关闭数据库，并且进入mount模式下进行备份。  
##### 全备
RMAN中backup database默认就是全备，即backup full database。全备表示所有曾经使用过的数据块都会被备份下来，不管它现在是否为空。
##### 增量备份
增量备份是相对于全备而言，增量备份只备份上一次备份以来所改变的数据块。增量备份需要显式指定incremental关键字。增量备份分为差异增量备份和累积增量备份。
##### 差异增量备份
是备份上级及同级备份以来所有变化的数据块，差异增量是默认增量备份方式。在lv1的差异增量备份中，RMAN备份自从在上一次lv1或者lv0备份后改变的数据块，如果lv0备份不可用，根据compatibility的不同，RMAN会有不同的做法：如果compatibility>=10.0.0，RMAN会备份自从数据文件创建以来所有改变了的数据块，否则，RMAN执行0级备份。图示如下：  
![differential](/images/differential.png)  
##### 累积增量备份
累积增量备份上级备份以来所有变化的块，如在lv1的累积增量备份中，它会备份上次lv0以后所有改变的数据块，在lv2的累积增量备份中，它会备份上次lv1以后所有改变的数据块，如果没有lv1，则会寻找lv0。图示如下：  
![cumulative](/images/cumulative.png)  
很明显，两种增量备份策略的差异在于，diff需要更少的备份空间，但是恢复时间更久，cumu需要的备份空间更多，但是恢复时间要少得多。如何选择，得综合两者考虑。同时，虽然存在2级增量备份，但是oracle不推荐使用2级增量备份。  
再者，0级增量和全备的区别在于：lv0可以用来增量恢复，而全备不可以，且lv0是增量备份基础。RMAN的默认设置里面，备份集是默认的，全备是默认的，增量备份中，差异增量备份是默认的。  
### 3. RMAN format
RMAN通过backup命令的<code>format</code>子句指定生成备份的路径和文件名。format参数如下表所示：  
<center>表1-1 RMAN Format格式说明</center>

|参数|含义|
|----|----|
|%a|指定数据库的激活ID
|%b|11g新加参数，定义没有任何目录路径的文件名，只能被set newname命令使用，或者用于建立图像副本备份
|%c|指定多路备份中备份片副本数量，如果没有使用多路备份，这个变量值是1(备份集)或者0(代理备份)
|%d|数据库名
|%D|当前日期天数，格式为DD
|%e|指定归档日志的sequence NO
|%f|指定绝对的文件数
|%F|提供唯一的和可重复的名称，包含了DBID，年月日等信息，这是系统留给控制文件自动备份用的格式，不是有效的用户指定格式
|%h|指定归档日志线程号(thread number)
|%I|指定DBID
|%M|指定当前日期月份，格式为MM
|%N|指定表空间名称，此参数仅用于备份数据文件或者镜像复制
|%n|指定数据库名称，右边用x字符填充至总长为8个字符，如DB_NAME为LINORA,使用此参数后为LINORAxx
|%p|表示备份集中的备份片数量，对于每个备份集，该值初始为1，在创建每个备份片时，该值增加1
|%s|指定备份集数量。该数量是控制文件中的计数器。该数值在控制文件的生命周期中唯一
|%t|表示备份集时间戳，这是根据从固定参考时间以来已经过去的秒数得出的4位字节值，结合%s可生成备份集唯一名称
|%T|指定时间格式，格式为YYYYMMDD
|%u|生产8位字符名称，该名称由备份集数量及备份集创建时间的压缩表示组成
|%U|默认的文件命名模式，在备份集中，%U表示%u_%p_%c组成，保证生成备份文件名唯一
|%Y|表示当前日期年份，格式为YYYY
|%%|转义字符，表示希望使用%字符
### 4. 控制文件及spfile自动备份
RMAN中，默认是不自动备份控制文件。通过以下命令可以开启控制文件及SPFILE的自动备份，且可以指定备份路径：
```
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/oradata/backup/%F.ctl';
```
同时，如果在归档模式下，只要有对控制文件内容有影响的数据库改变，都会导致控制文件自动备份，如创建表空间，创建表等。如果数据库版本是10g，则可立刻从alert日志看出控制文件的自动备份，但是在11gR2中，引进了控制文件自动备份延迟创建的特性。即使autobackup开启，在数据结构改变时不会立即看到控制文件的备份，而是过一段时间才能看到，这是为了改变性能而引进的，防止用户在一个脚本中多次对数据库结构的变化而创建多个控制文件备份。它是通过MMON后台完成的，因此，11gr2在alert日志中是看不到控制文件的自动备份信息的，而是写在m000 trace文件中。至于延长多久，是有隐含参数<code>_controlfile_autobackup_delay</code>来控制的，默认：300s。
```
#m000 trace文件部分信息
Starting control autobackup

*** 2014-06-01 09:06:48.030
Control autobackup written to DISK device
        handle '/oradata/backup/c-3385851293-20140601-01.ctl'
#查看隐含参数
SQL> set linesize 200
SQL> col KSPPINM for a30
SQL> col KSPPSTVL for a10
SQL> col KSPPDESC for a100
SQL> SELECT ksppinm, ksppstvl, ksppdesc
  2    FROM x$ksppi x, x$ksppcv y
  3   WHERE x.indx = y.indx
  4     AND ksppinm = '_controlfile_autobackup_delay';
KSPPINM                        KSPPSTVL   KSPPDESC
------------------------------ ---------- ---------------------------------------------------------------
_controlfile_autobackup_delay  300        time delay (in seconds) for performing controlfile autobackups
```
同时，如果设定<code>AUTOBACKUP ON</code>，则在任何一个backup命令成功后，都会备份控制文件及SPFILE。
```
Starting Control File and SPFILE Autobackup at 2014-06-01 09:24:07
piece handle=/oradata/backup/c-3385851293-20140601-04.ctl comment=NONE
Finished Control File and SPFILE Autobackup at 2014-06-01 09:24:10
```
如果备份的数据文件是system，即datafile 1，那么不管自动备份开启与否，RMAN都将自动备份控制文件与SPFILE。
### 5. RMAN备份示例
```
run{
allocate channel c1 type disk format '/oradata/backup/lv0_%d_%T_%s';
#分配通道c1，类型为disk，指定文件名和备份路径
backup incremental level=0 database
#进行0级增量备份
include current controlfile
#包含当前控制文件
plus archivelog  delete all input;
#包含归档日志，且将备份完的归档删除
release channel c1;
#释放通道
}

allocated channel: c1
channel c1: SID=143 device type=DISK


Starting backup at 2014-06-01 00:17:33
current log archived #归档当前在线日志
channel c1: starting archived log backup set
channel c1: specifying archived log(s) in backup set
input archived log thread=1 sequence=38 RECID=110 STAMP=849053853
channel c1: starting piece 1 at 2014-06-01 00:17:33
channel c1: finished piece 1 at 2014-06-01 00:17:34
piece handle=/oradata/backup/lv0_LINORA_20140601_81 tag=TAG20140601T001733 comment=NONE
#产生的备份文件名称
channel c1: backup set complete, elapsed time: 00:00:01
channel c1: deleting archived log(s)
archived log file name=/oradata/arch/1_38_842803310.arc RECID=110 STAMP=849053853
Finished backup at 2014-06-01 00:17:34
#完成备份数据文件前的归档日志备份

Starting backup at 2014-06-01 00:17:34
channel c1: starting incremental level 0 datafile backup set
channel c1: specifying datafile(s) in backup set
input datafile file number=00001 name=/oradata/datafile/linora/system01.dbf
input datafile file number=00002 name=/oradata/datafile/linora/sysaux01.dbf
input datafile file number=00003 name=/oradata/datafile/linora/undotbs01.dbf
input datafile file number=00005 name=/oradata/datafile/linora/fung01.dbf
input datafile file number=00004 name=/oradata/datafile/linora/users01.dbf
input datafile file number=00006 name=/oradata/datafile/linora/undotbs02.dbf
channel c1: starting piece 1 at 2014-06-01 00:17:35
channel c1: finished piece 1 at 2014-06-01 00:18:40
piece handle=/oradata/backup/lv0_LINORA_20140601_82 tag=TAG20140601T001734 comment=NONE
channel c1: backup set complete, elapsed time: 00:01:05
#完成数据文件备份
channel c1: starting incremental level 0 datafile backup set
channel c1: specifying datafile(s) in backup set
including current control file in backup set
including current SPFILE in backup set
channel c1: starting piece 1 at 2014-06-01 00:18:41
channel c1: finished piece 1 at 2014-06-01 00:18:42
piece handle=/oradata/backup/lv0_LINORA_20140601_83 tag=TAG20140601T001734 comment=NONE
channel c1: backup set complete, elapsed time: 00:00:01
Finished backup at 2014-06-01 00:18:42
#完成控制文件的备份，同时spfile也包含在此备份集中

Starting backup at 2014-06-01 00:18:42
current log archived #归档当前日志，目的在于备份在此期间内的日志变化
channel c1: starting archived log backup set
channel c1: specifying archived log(s) in backup set
input archived log thread=1 sequence=39 RECID=111 STAMP=849053922
channel c1: starting piece 1 at 2014-06-01 00:18:42
channel c1: finished piece 1 at 2014-06-01 00:18:43
piece handle=/oradata/backup/lv0_LINORA_20140601_84 tag=TAG20140601T001842 comment=NONE
channel c1: backup set complete, elapsed time: 00:00:01
channel c1: deleting archived log(s)
archived log file name=/oradata/arch/1_39_842803310.arc RECID=111 STAMP=849053922
Finished backup at 2014-06-01 00:18:43

released channel: c1
```
### 6. RMAN维护
##### 确认过期/失效备份
首先对RMAN备份集进行<code>crosscheck</code>，这个命令会更新catalog信息，以便RMAN得到的是最新的真实数据，例如在OS级别删除了备份文件，则需要crosscheck去同步catalog信息。<code>report obsolete</code>跟RMAN备份策略的冗余有关。
```
RMAN> crosscheck backup;
RMAN> report obsolete;
RMAN> delete obsolete;
RMAN> delete expired backup;
```
##### RMAN list
RMAN的<code>list</code>命令能查看备份状态
```
RMAN> list backup;
RMAN> list backup summary;
RMAN> list backup by file;
RMAN> list backup of database;
RMAN> list backup of controlfile;
RMAN> list backup of archivelog all; 
RMAN> list backup of spfile;
RMAN> list expired backup;
RMAN> list backupset;
RMAN> list incarnation;
```
##### RMAN备份完整性检测
使用<code>backup validate</code>对备份数据进行完整性检测，跟备份过程完全相同，但不会执行真正的备份。如果要检测逻辑上的corruption，则添加<code>check logical</code>参数。
```
RMAN> backup validate archivelog all;

Starting backup at 2014-06-01 10:10:03
current log archived
using channel ORA_DISK_1
channel ORA_DISK_1: starting archived log backup set
channel ORA_DISK_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=45 RECID=117 STAMP=849089403
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Archived Logs
=====================
Thrd Seq     Status Blocks Failing Blocks Examined Name
---- ------- ------ -------------- --------------- ---------------
1    45      OK     0              4082            /oradata/arch/1_45_842803310.arc
Finished backup at 2014-06-01 10:10:04

RMAN> backup validate check logical archivelog all;

Starting backup at 2014-06-01 10:10:25
current log archived
using channel ORA_DISK_1
channel ORA_DISK_1: starting archived log backup set
channel ORA_DISK_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=45 RECID=117 STAMP=849089403
input archived log thread=1 sequence=46 RECID=118 STAMP=849089425
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Archived Logs
=====================
Thrd Seq     Status Blocks Failing Blocks Examined Name
---- ------- ------ -------------- --------------- ---------------
1    45      OK     0              4082            /oradata/arch/1_45_842803310.arc
1    46      OK     0              1               /oradata/arch/1_46_842803310.arc
Finished backup at 2014-06-01 10:10:26
```
如果备份文件中存在corruption，可以通过查询<code>V$DATABASE_BLOCK_CORRUPTION</code>获得坏块位置。下面模拟一个物理坏块：
```
[oracle@linora:/oradata/datafile/linora]$ dd of=/oradata/datafile/linora/test01.dbf \
bs=8192 conv=notrunc seek=15 <<! 
> abc
> !
0+1 records in
0+1 records out
4 bytes (4 B) copied, 0.000733863 s, 5.5 kB/s
RMAN> validate datafile 8;

Starting validate at 2014-06-01 10:25:37
using channel ORA_DISK_1
channel ORA_DISK_1: starting validation of datafile
channel ORA_DISK_1: specifying datafile(s) for validation
input datafile file number=00008 name=/oradata/datafile/linora/test01.dbf
channel ORA_DISK_1: validation complete, elapsed time: 00:00:03
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
8    FAILED 0              1153         1280            1510913   
  File Name: /oradata/datafile/linora/test01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              0               
  Index      0              0               
  Other      1              127             

validate found one or more corrupt blocks
See trace file /u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_5245.trc for details
Finished validate at 2014-06-01 10:25:40
SQL> select * from V$DATABASE_BLOCK_CORRUPTION;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO
---------- ---------- ---------- ------------------ ---------
         8         15          1                  0 CORRUPT
```
查询结果跟我们预期的相同，都是在数据文件8，block 15上坏块。对于坏块的检测和处理，请参照前文[Detect and Correct Corruption in Oracle](http://www.oraclema.com/oracle/detect-and-correct-corruption-in-oracle.html)。
### 7. 查看RMAN备份进度
```
#查看RMAN备份进度
SQL> select sid,serial#,context,sofar,totalwork,
  2  round(sofar/totalwork*100,2) "%_COMPLETE"
  3  from v$session_longops
  4  where opname like '%RMAN%'
  5  and opname not like '%aggregate%'
  6  and totalwork !=0
  7  and sofar<>totalwork
  8  /

       SID    SERIAL#    CONTEXT      SOFAR  TOTALWORK %_COMPLETE
---------- ---------- ---------- ---------- ---------- ----------
       148        153          1     142204     250368       56.8
```
```
SQL> col CLIENT_INFO for a10
SQL> col OPNAME for a20
SQL> col MESSAGE for a30
SQL> col SPID for a10
SQL> set linesize 200
SQL> select s.client_info,
sl.opname,
sl.message,
sl.sid, sl.serial#, p.spid,
sl.sofar, sl.totalwork,
round(sl.sofar/sl.totalwork*100,2) "% Complete"
from v$session_longops sl, v$session s, v$process p
where p.addr = s.paddr
and sl.sid=s.sid
and sl.serial#=s.serial#
and opname LIKE 'RMAN%'
and opname NOT LIKE '%aggregate%'
and totalwork != 0
and sofar <> totalwork;

CLIENT_INF OPNAME               MESSAGE                               SID    SERIAL# SPID            SOFAR  TOTALWORK % Complete
---------- -------------------- ------------------------------ ---------- ---------- ---------- ---------- ---------- ----------
rman chann RMAN: full datafile  RMAN: full datafile backup: Se         10         75 5425           249150     250368      99.51
el=ch00    backup               t Count 129: 249150 out of 250
                                368 Blocks done
```
Reference：  
[Oracle® Database Backup and Recovery User's Guide 11g Release 2 (11.2)](http://docs.oracle.com/cd/E11882_01/backup.112/e10642/toc.htm)















