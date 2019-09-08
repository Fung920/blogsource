---
title: RMAN Compression And Unused Block Compression
categories: oracle
comments: false
date: 2019-08-08 00:05:23
tags: rman
---
最近在检查备份的时候发现一个新库，在磁带上备份占用了将近1个T，而备份到本地磁盘才使用了200G。
从v$segments看，空间占用大概200G左右，但是分配了仅1个T的空间。为此，专门查看了MOS文档，里面有几篇文档提及这个问题。

<!--more-->

# 1. RMAN压缩的类型
RMAN默认提供以下三种类型的备份压缩。
其中，从10.2开始，null compression和Unused block compression默认自动会进行，不需要其他指定。但是，在一些情况下，unused block compression会失效。
## 1.1 Null compression
当以备份集方式备份数据文件的时候，RMAN不会去备份没有被分配的数据块，例如，在一个数据文件中，分配了100M空间，但仅仅使用了50M，rman只会备份这个50M的块。
## 1.2 Unused block compression
从10.2开始，rman会跳过当前没有包含数据的块，这个称之为unused block compression, 例如，当前数据文件100M，包含了50M数据，当用户drop包含25M的表时候，rman只会备份剩下的25M的空间。
即是说，null compression只会备份使用了的包括使用过的块；而unused block compression只备份了当前含有数据的数据块。因此unused block compress比null compression要节省空间跟时间。

Unused block compression适用条件：
* `COMPATIBLE`必须是10.2及以上
* 没有定义还原点
* 数据文件为本地管理
* 数据文件必须是全备或者0级增量备份，且备份以备份集形式存在
* 备份集是存放到磁盘或者以Oracle Secure Backup备份到磁带，__即其他第三方备份软件备份到磁带是不会具有这个功能__
* 仅支持企业版

这也是为什么第三方备份软件备份到磁带上空间占用量要远高于磁盘了。

## 1.3 Binary compression
二进制压缩需要在rman备份命令中指定`AS COMPRESSED`。以这种形式的压缩备份成为二进制压缩。
在进行二进制压缩备份的时候，CPU会有开销，意味着备份时间更长，同时restore的时候，时间也比没有压缩的还原时间要长。
从11.2开始，rman二进制压缩有以下三个级别：
* LOW - 最小的压缩比例和占用最少的CPU
* MEDIUM - 同时兼顾CPU跟压缩比，推荐使用
* HIGH - 最佳的压缩比，同时CPU开销会更大，使用于磁盘空间不足，但CPU比较空闲，或者需要通过网络传输，但网络资源比较紧张的情况

通过查询`V$RMAN_COMPRESSION_ALGORITHM`可以看到Oracle支持的压缩算法：

```sql
SQL> select ALGORITHM_NAME, ALGORITHM_DESCRIPTION, ALGORITHM_COMPATIBILITY from V$RMAN_COMPRESSION_ALGORITHM ;

ALGORITHM_NAME  ALGORITHM_DESCRIPTION                                        ALGORITHM_COMPATIB
--------------- ------------------------------------------------------------ ------------------
BZIP2           good compression ratio                                       9.2.0.0.0
BASIC           good compression ratio                                       9.2.0.0.0
LOW             maximum possible compression speed                           11.2.0.0.0
ZLIB            balance between speed and compression ratio                  11.0.0.0.0
MEDIUM          balance between speed and compression ratio                  11.0.0.0.0
HIGH            maximum possible compression ratio                           11.2.0.0.0
```

通过rman以下命令可以改变默认的压缩类型：

```sql
RMAN> CONFIGURE COMPRESSION ALGORITHM '<alg_name>';
```

# 2. Undo块的优化
从11g开始，rman开始对undo块进行优化，在不需要进行recovery的undo block上，即已经提交的事务的undo block，这些undo block将会被rman排除掉。
同时undo的备份优化需满足以下几个条件：

* 是备份集的备份
* 全备或者是0级增量备份
* 不是validate的rman备份有效性验证
* 备份片的版本是11.1及以上
* 用户没有禁用undo optimization： `_undo_block_compression = FALSE`
* 备份集是写入磁盘或者OSB的磁带
* 没有还原点

# 3. Example

* 数据量大小：

```sql
SQL> select sum(bytes/1024/1024/1024) from dba_segments;

SUM(BYTES/1024/1024/1024)
-------------------------
3.20123291
```

* RMAN基于磁盘的备份：

```sql
RMAN> backup full database format '/home/oracle/backup/full_%U.bak';
```

* 磁盘备份的大小：

```sql
[oracle@db2srv:/home/oracle/backup]$ du -sh full_0*
1.9G	full_01u9pved_1_1.bak
9.7M	full_02u9pvfi_1_1.bak
```

* 模拟磁带备份

```sql
RMAN> run {
allocate channel x1 type 'sbt_tape'
parms="SBT_LIBRARY=oracle.disksbt,
ENV=(BACKUP_DIR=/home/oracle/backup)";
BACKUP full database format 'type_%U.bak';
}
```

* 磁带备份大小

```sql
[oracle@db2srv:/home/oracle/backup]$ du -sh type_0*
9.3G	type_08u9q0ma_1_1.bak
17M	type_09u9q0o2_1_1.bak
```

可以看到，磁带的备份是并没有排除空块的。
从`v$backup_datafile`视图中，可以看出数据文件是否是full scan(%READ is 100):

```sql
select file# fno, used_change_tracking BCT,
datafile_blocks BLKS,
block_size blksz, blocks_read READ,
UNDO_OPTIMIZED,
round((blocks_read/datafile_blocks) * 100,2) "%READ",
blocks WRTN, round((blocks/datafile_blocks)*100,2) "%WRTN"
from v$backup_datafile
--where completion_time between
--      to_date('25-jan-09 01:30:00', 'dd-mon-rr hh24:mi:ss') and
--      to_date('26-jan-09 12:35:26', 'dd-mon-rr hh24:mi:ss')
order by file#;
```




</br>
__Reference__:
[**Why is RMAN not using Unused Block Compression during backup? (Doc ID 798844.1)**](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=322241885364989&id=798844.1&displayIndex=16&_afrWindowMode=0&_adf.ctrl-state=12zs90rvy4_171)
[A Complete Understanding of RMAN Compression (Doc ID 563427.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=322717803996859&id=563427.1&displayIndex=18&_afrWindowMode=0&_adf.ctrl-state=12zs90rvy4_283#GOAL)
[Why an SBT backup MAY take longer and create larger backuppieces than a DISK backup. (Doc ID 1349492.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=366882990480648&parent=WIDGET_RECENT_DB&id=1349492.1&_afrWindowMode=0&_adf.ctrl-state=wculeptf1_65)

__EOF__
