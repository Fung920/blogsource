---
layout: post
title: "DB_FILE_MULTIBLOCK_READ_COUNT"
date: 2014-08-21 11:13:08
comments: false
categories: oracle
tags: tuning
keywords: db_file_multiblock_read_count,db file scattered read,direct path read
description: db_file_multiblock_read_count参数对数据库性能的影响
---
db_file_multiblock_read_count参数代表Oracle在执行table full scan、index full scan或者index fast full scan时每次IO操作可以读取的数据块的数量。
<!--more-->
### 1. 相关等待事件
当数据的访问方式符合上面三种的时候，为了保障性能，尽量一次读取多个块，即Multi Block I/O。每次执行Multi Block I/O，都会等待物理I/O结束，此时产生等待db file scattered read事件。11g中有一个新特性，全表扫描可以通过直接路径读(direct path read)的方式来执行。
<li>db file scattered read 等待事件：是由于多数据块读操作产生的，当检索数据时从磁盘上读数据到内存中，一次I/O读取多个数据块，而数据块在内存中是分散分布并不是连续的，当数据块读取到内存这个过程中会产生'db file scattered read'事件</li>
<li>direct path read 等待事件：11g中，大表全表扫描时数据块不经过sga而直接读取到会话私有PGA中，一般是PGA的sort area区。在这个过程中将会发生'direct path read'等待事件</li>
#### 1.1 direct path read
```
SYS@linora> select * from v$version where rownum<2;
BANNER
--------------------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production

SYS@linora> alter session set db_file_multiblock_read_count=16;
Session altered.
SYS@linora> alter session set events '10046 trace name context forever, level 12';
Session altered.
SYS@linora> select /*+ FULL(t) */ count(*) from sys.source$ t;
  COUNT(*)
----------
    611684
SYS@linora> alter session set events '10046 trace name context off';
Session altered.
SYS@linora> col value for a80
SYS@linora> select value from v$diag_info where name='Default Trace File';
VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_4222.trc
```
部分trace文件：
```
select /*+ FULL(t) */ count(*) from sys.source$ t
END OF STMT
...
WAIT # 140711463500848: nam='direct path read' ela= 185 file number=1 first dba=1505 block cnt=7 obj#=224 tim=1408604487942484
WAIT # 140711463500848: nam='direct path read' ela= 109 file number=1 first dba=8168 block cnt=16 obj#=224 tim=1408604487942701
...
WAIT # 140711463500848: nam='direct path read' ela= 100 file number=1 first dba=10272 block cnt=8 obj#=224 tim=1408604487948483
WAIT # 140711463500848: nam='direct path read' ela= 80 file number=1 first dba=10312 block cnt=8 obj#=224 tim=1408604487949419
WAIT # 140711463500848: nam='direct path read' ela= 262 file number=1 first dba=10360 block cnt=8 obj#=224 tim=1408604487950114
WAIT # 140711463500848: nam='direct path read' ela= 130 file number=1 first dba=10368 block cnt=16 obj#=224 tim=1408604487951202
WAIT # 140711463500848: nam='direct path read' ela= 113 file number=1 first dba=10408 block cnt=16 obj#=224 tim=1408604487952832
...
STAT # 140711463500848 id=2 cnt=611684 pid=1 pos=1 obj=224 op='TABLE ACCESS FULL SOURCE$ (cr=8058 pr=8055 pw=0 time=2716063 us cost=1773 size=0 card=611677)'
```
从以上可以看到等待事件为'direct path read'，访问数据的方式为'TABLE ACCESS FULL SOURCE$'，且每次最多只能读取16个块(block cnt=16)。
#### 1.2 db file scattered read
在11g中，全表扫描已经使用direct path read，但可以设置隐含参数<code>_serial_direct_read</code>来禁止串行直接路径读，或者设置10949事件屏蔽11g的这个新特性，返回11g以前的模式上。
```
SYS@linora> alter session set events '10949 trace name context forever, level 1';
Session altered.
SYS@linora> alter session set events '10046 trace name context forever, level 12';
Session altered.
SYS@linora> alter session set db_file_multiblock_read_count=16;
Session altered.
SYS@linora> select /*+ FULL(t) */ count(*) from sys.source$ t;
  COUNT(*)
----------
    611684
SYS@linora> alter session set events '10046 trace name context off';
Session altered.
SYS@linora> col value for a80
SYS@linora> select value from v$diag_info where name='Default Trace File';
VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_4502.trc
```
部分trace文件：
```
select /*+ FULL(t) */ count(*) from sys.source$ t
END OF STMT
WAIT # 140430667434504: nam='db file scattered read' ela= 83 file#=1 block#=1505 blocks=7 obj#=224 tim=1408605703478759
WAIT # 140430667434504: nam='db file scattered read' ela= 67 file#=1 block#=8168 blocks=8 obj#=224 tim=1408605703479239
WAIT # 140430667434504: nam='db file scattered read' ela= 44 file#=1 block#=8176 blocks=8 obj#=224 tim=1408605703479634
...
WAIT # 140430667434504: nam='db file scattered read' ela= 33 file#=1 block#=10864 blocks=16 obj#=224 tim=1408605703488577
WAIT # 140430667434504: nam='db file scattered read' ela= 34 file#=1 block#=16128 blocks=16 obj#=224 tim=1408605703489184
...
STAT # 140430667434504 id=2 cnt=611684 pid=1 pos=1 obj=224 op='TABLE ACCESS FULL SOURCE$ (cr=8064 pr=8055 pw=0 time=2097213 us cost=1773 size=0 card=611677)'
```
从以上可以看到等待事件变为'db file scattered read'，访问数据的方式为'TABLE ACCESS FULL SOURCE$'，且每次最多只能读取16个块(blocks=16)。  
对隐含参数的描述：
```
SYS@linora> set linesize 120
SYS@linora> col name for a30
SYS@linora> col value for a10
SYS@linora> col PDESC for a50
SYS@linora> SELECT x.ksppinm NAME, y.ksppstvl VALUE, x.KSPPDESC PDESC
  2  FROM SYS.x$ksppi x, SYS.x$ksppcv y
  3  WHERE x.indx = y.indx
  4  AND x.ksppinm LIKE '%&par%';
Enter value for par: _serial_direct_read
old   4: AND x.ksppinm LIKE '%&par%'
new   4: AND x.ksppinm LIKE '%_serial_direct_read%'

NAME                           VALUE      PDESC
------------------------------ ---------- --------------------------------------------------
_serial_direct_read            auto       enable direct read in serial
```
该隐含参数是动态参数，我们可以通过alter system set方式修改
<li>_serial_direct_read = FALSE，禁用direct path read</li>
<li>_serial_direct_read = TURE，重新启用direct path read</li>
### 2. db_file_multiblock_read_count最大值
```
FUNG@linora> show parameter db_block_size
NAME                                 TYPE        VALUE
------------------------------------ ----------- -------------------
db_block_size                        integer     8192
```
将<code>db_file_multiblock_read_count</code>设置成无限大
```
SYS@linora> col value for a10
SYS@linora> alter session set db_file_multiblock_read_count = 9999999;
Session altered.

SYS@linora> select value from v$parameter where name = 'db_file_multiblock_read_count';
VALUE
----------
4096
```
可以看到，本机的blocksize=8K，db_file_multiblock_read_count=4096，因此，在此OS上，理论最大一次可读取8K*4096=32M数据。理论上，最大的mbrc和系统IO有如下关系：
```
max(db_file_multiblock_read_count)=max os io size/db_block_size
```
但是，由于Oracle的一次IO不能跨extent，因此，Oracle的IO还要受到extent的影响，本例中extent均为默认的1M大小。即Oracle一次IO最大能扫描128块(1M)数据。
```
FUNG@linora> alter session set events '10949 trace name context forever, level 1';
Session altered.
FUNG@linora> alter session set db_file_multiblock_read_count=99999;
Session altered.
FUNG@linora> show parameter db_file_multiblock_read_count
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_file_multiblock_read_count        integer     4096
FUNG@linora> alter session set events '10046 trace name context forever, level 12';
Session altered.
FUNG@linora> select /*+ FULL(t) */ count(*) from sys.source$ t;
FUNG@linora> alter session set events '10046 trace name context off';
```
trace文件部分信息：
```
WAIT # 139908117497232: nam='db file scattered read' ela= 546 file#=1 block#=77056 blocks=128 obj#=224 tim=1408611829156495
WAIT # 139908117497232: nam='db file scattered read' ela= 581 file#=1 block#=78336 blocks=128 obj#=224 tim=1408611829159201
WAIT # 139908117497232: nam='db file scattered read' ela= 1274 file#=1 block#=78464 blocks=128 obj#=224 tim=1408611829162572
WAIT # 139908117497232: nam='db file scattered read' ela= 350 file#=1 block#=78592 blocks=128 obj#=224 tim=1408611829164950
WAIT # 139908117497232: nam='db file scattered read' ela= 664 file#=1 block#=78848 blocks=128 obj#=224 tim=1408611829167615
```
这里确实显示最大读取了128个块。但如果使用11G新特性，全表扫描用'direct path read'，会显示不同结果：
```
WAIT # 140535124395624: nam='direct path read' ela= 3269 file number=1 first dba=83712 block cnt=1024 obj#=224 tim=1408612856901580
```
此时最大的读取块为1024个，即8M，估计11g有其他限制，以后有时间再研究。
### 3. 不同大小对性能的影响
```
FUNG@linora> create table t as select * from dba_objects;
Table created.
FUNG@linora> create index i_owner on t(owner);
Index created.
FUNG@linora> analyze table t compute statistics for table
for all indexes
for all indexed columns;
FUNG@linora> alter system flush buffer_cache;
System altered.
FUNG@linora> set autotrace traceonly explain
FUNG@linora> alter session set db_file_multiblock_read_count=8;
Session altered.
FUNG@linora> select * from t where owner='PUBLIC';
33346 rows selected.

Execution Plan
----------------------------------------------------------
Plan hash value: 1601196873

--------------------------------------------------------------------------
| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |      | 33346 |  3289K|   339   (1)| 00:00:05 |
|*  1 |  TABLE ACCESS FULL| T    | 33346 |  3289K|   339   (1)| 00:00:05 |
--------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("OWNER"='PUBLIC')
```
将<code>db_file_multiblock_read_count</code>设置为64：
```
FUNG@linora> alter system flush buffer_cache;
System altered.
FUNG@linora> alter session set db_file_multiblock_read_count=64;
Session altered.
FUNG@linora> select * from t where owner='PUBLIC';
33346 rows selected.

Execution Plan
----------------------------------------------------------
Plan hash value: 1601196873
--------------------------------------------------------------------------
| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |      | 33346 |  3289K|   226   (1)| 00:00:03 |
|*  1 |  TABLE ACCESS FULL| T    | 33346 |  3289K|   226   (1)| 00:00:03 |
--------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("OWNER"='PUBLIC')
```
可以看到在全表扫描时，随着mbrc的增大，CPU cost从339降到226。
### 4. 总结
在Oracle10gR2之后的版本（10gR2和11g）中，Oracle数据库已经可以根据系统的IO能力以及Buffer Cache的大小来动态调整该参数值，Oracle建议不要显式设置该参数值。  
在OLAP系统中，如果确实有大量需要全表扫描的SQL，可以考虑设置比较大一点。  


Reference:  
[深入解析Oracle](http://www.eygle.com/archives/2013/12/hforacle_release.html)  
[Oracle 11g全表扫描以Direct Path Read方式执行](http://www.eygle.com/archives/2012/05/oracle_11g_direct_path_read.html)  
