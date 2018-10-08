---
layout: post
title: "BBED Demo"
date: 2014-09-05 10:46:13
comments: false
categories: oracle
tags: bbed
keywords: bbed
description: bbed模拟修改数据块
---
BBED功能有限，对很多类型的数据块都不支持。BBED虽然支持在open状态下修改数据块，但建议先干净的关闭数据库在进行BBED修改。正常关闭数据库能避开检查点进程覆盖掉bbed修改的块缓存。
<!--more-->
### 1. 修改数据
```
FUNG@linora> select * from test;
ID         NAME
---------- --------------------
002        Kong
001        Fung
```
假设修改id=002的name为kyun。首先确定id=2所在的dba：
```
SYS@linora> SELECT DBMS_ROWID.ROWID_RELATIVE_FNO(rowid) "FILE",
  2  DBMS_ROWID.ROWID_BLOCK_NUMBER(rowid) "BLOCK",
  3  DBMS_ROWID.ROWID_ROW_NUMBER(rowid) "ROW",
  4  id,name
  5  FROM fung.test;

      FILE      BLOCK        ROW ID         NAME
---------- ---------- ---------- ---------- --------------------
         5        212          0 002        Kyun
         5        213          0 001        Fung
```
首先定位数据所在dba
```
BBED> set dba 5,212
        DBA             0x014000d4 (20971732 5,212)
BBED> find /c Kong
 File: /oradata/datafile/linora/fung01.dbf (5)
 Block: 212              Offsets: 8174 to 8191           Dba:0x014000d4
------------------------------------------------------------------------
 4b6f6e67 2c000201 32044b6f 6e670106 7319 

 <32 bytes per line>
BBED> d /v dba 5,212 offset 8174 count 64
 File: /oradata/datafile/linora/fung01.dbf (5)
 Block: 212     Offsets: 8174 to 8191  Dba:0x014000d4
-------------------------------------------------------
 4b6f6e67 2c000201 32044b6f 6e670106 l Kong,...2.Kong..
 7319                                l s.

 <16 bytes per line>
```
开始修改数据:
```
BBED> m /c kyun dba 5,212 offset 8174
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /oradata/datafile/linora/fung01.dbf (5)
 Block: 212              Offsets: 8173 to 8191           Dba:0x014000d4
------------------------------------------------------------------------
 6b79756e 672c0002 0132044b 6f6e6701 067319 

 <32 bytes per line>
BBED> d /v
 File: /oradata/datafile/linora/fung01.dbf (5)
 Block: 212     Offsets: 8174 to 8191  Dba:0x014000d4
-------------------------------------------------------
 6b79756e 2c000201 32044b6f 6e670106 l kyun,...2.Kong..
 7119                                l q.

 <16 bytes per line>
```
此时数据库级别是看不到的，需要sum验证一下，同时刷新库缓存：
```
BBED> sum apply
Check value for File 5, Block 212:
current = 0x04bb, required = 0x04bb
SYS@linora> alter system flush buffer_cache;
System altered.
SYS@linora> select * from fung.test;
ID         NAME
---------- --------------------
002        Kyun
001        Fung
```
### 2. 找回删除的数据
在Oracle中，delete操作并不是真正删除了该行数据，而是将被删除的行标记为删除状态，
```
SYS@linora> alter system dump datafile 5 block 212;
System altered.
--dump trace内容
block_row_dump:
tab 0, row 0, @0x1f82
tl: 12 fb: --H-FL-- lb: 0x0  cc: 2
col  0: [ 3]  30 30 32
col  1: [ 4]  4b 79 75 6e
```
删除表中数据：
```
SYS@linora> delete from fung.test;
2 rows deleted.
SYS@linora> commit;
Commit complete.
SYS@linora> alter system flush buffer_cache;
System altered.
SYS@linora> alter system dump datafile 5 block 212;
System altered.
--dump trace内容
block_row_dump:
tab 0, row 0, @0x1f82
tl: 2 fb: --HDFL-- lb: 0x1
```
由两次的dump文件内容可以看到，被删除了的数据会在fb栏位添加一个D的标识符。如果一个row 没有被删除，那么它就具有上面的3个属性，即Flag 表示为：--H-FL--. 这里的字母分别代表属性的首字母。其对应的值：32 + 8 + 4 =44 or 0x2c。如果一个row 被delete了，那么row flag 就会更新，bitmask 里的deleted 被设置为16. 此时row flag 为： 32 + 16 + 8 + 4 = 60 or 0x3c。  
如果需要将被删除的数据找回来，只需要修改3c为2c即可：
```
BBED> set dba 5,212
        DBA             0x014000d4 (20971732 5,212)
--查看此块包含几行记录
BBED> map
 File: /oradata/datafile/linora/fung01.dbf (5)
 Block: 212                                   Dba:0x014000d4
------------------------------------------------------------
 KTB Data Block (Table/Cluster)
 struct kcbh, 20 bytes                      @0       
 struct ktbbh, 72 bytes                     @20      
 struct kdbh, 14 bytes                      @100     
 struct kdbt[1], 4 bytes                    @114     
--只有一行数据
 sb2 kdbr[1]                                @118     
 ub1 freespace[8046]                        @120     
 ub1 rowdata[22]                            @8166    
 ub4 tailchk                                @8188    
BBED> p *kdbr[0]
rowdata[0]
----------
ub1 rowdata[0]                              @8166     0x3c
```
修改offset 8166中的3c为2c，然后验证被删除的数据是否有被找回：
```
BBED> m /x 2c dba 5,212 offset 8166
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /oradata/datafile/linora/fung01.dbf (5)
 Block: 212              Offsets: 8166 to 8191           Dba:0x014000d4
------------------------------------------------------------------------
 2c010203 30303204 4b79756e 2c000201 32044b6f 6e670106 ade7 
 <32 bytes per line>
BBED> sum apply
Check value for File 5, Block 212:
current = 0x5f63, required = 0x5f63
SYS@linora> alter system flush buffer_cache;
System altered.
SYS@linora> select * from fung.test;
ID         NAME
---------- --------------------
002        Kyun
```
### 3. 归档缺失的情况下增进文件头SCN
在数据文件恢复的时候，如果缺失部分归档日志，此时数据文件因为SCN值跟控制文件记录的SCN不一致，而导致无法Online，我们可以通过BBED修改文件头的SCN值去人为的控制，但会丢失部分数据。下面模拟删除一个数据文件，同时删除所有归档，然后recovery，但由于缺少归档，recovery肯定会失败，此时可以使用BBED修改文件头SCN进行人为的同步。
```
[oracle@linora:/oradata/arch]$ rm -rf /oradata/datafile/linora/test01.dbf
SYS@linora> startup force
RMAN> restore datafile '/oradata/datafile/linora/test01.dbf';  
RMAN> recover datafile '/oradata/datafile/linora/test01.dbf';
Starting recover at 2014-09-10 17:01:54
using channel ORA_DISK_1
starting media recovery
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 09/10/2014 17:01:54
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 194 and starting SCN of 3631541 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 193 and starting SCN of 3631538 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 192 and starting SCN of 3631535 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 191 and starting SCN of 3631234 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 190 and starting SCN of 3606559 found to restore
```
分别查看控制文件和数据文件头SCN：
```
--控制文件SCN
SYS@linora> select file#,checkpoint_change# from v$datafile;
     FILE# CHECKPOINT_CHANGE#
---------- ------------------
         1            3653096
         2            3653096
         3            3653096
         4            3653096
         5            3653096
         6            3653096
         7            3653096
         8            3653096
         9            3653096
        10            3653096
        11            3653096
11 rows selected.
--数据文件头SCN

SYS@linora> select file#,checkpoint_change# from v$datafile_header;
     FILE# CHECKPOINT_CHANGE#
---------- ------------------
         1            3653096
         2            3653096
         3            3653096
         4            3653096
         5            3653096
         6            3653096
         7            3653096
         8            3653096
         9            3653096
        10            3653096
        11            3606566
11 rows selected.
SYS@linora> select change# from v$recover_file; 
   CHANGE#
----------
   3606566
```
可以确定，file 11的SCN不一致，但又缺少归档，因此，media recovery无法进行。文件头信息存储在数据文件的第一个块内，在前文[BBED Structure](/bbed-structure.html)，块头仅有一个结构--kcvfh，数据文件是否与其他数据文件同步，Oracle会考虑到此结构中的四个属性：
<li>kscnbas(offset 484)--SCN of last change to the file</li>
<li>kcvcptim(offset 492)--Time of last change to the file</li>
<li>kcvfhcpc(offset 140)--Checkpoint count,v$datafile_header.CHECKPOINT_COUNT，数据文件发生Checkpoint的次数</li>
<li>kcvfhccc(offset 148)--控制文件检查点次数</li>
使用bbed分别对比1号文件和11号文件的上述四个属性值：  
##### 1号文件
```
BBED> set dba 1,1
        DBA             0x00400001 (4194305 1,1)
BBED> p kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484     
      ub4 kscnbas                           @484      0x0037bde8--检查点SCN
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x3322ed7b--检查点时间
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x000000c3
         ub4 kcrbabno                       @504      0x00000561
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00
BBED> p kcvfhcpc
ub4 kcvfhcpc                                @140      0x00000287--数据文件ckpt次数
BBED> p kcvfhccc
ub4 kcvfhccc                                @148      0x00000286--控制文件ckpt次数
```
##### 11号文件
```
BBED> set dba 11,1
        DBA             0x02c00001 (46137345 11,1)
BBED> p kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484     
      ub4 kscnbas                           @484      0x00370826
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x3322df58
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x000000be
         ub4 kcrbabno                       @504      0x00000002
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00
BBED> p kcvfhcpc
ub4 kcvfhcpc                                @140      0x00000044
BBED> p kcvfhccc
ub4 kcvfhccc                                @148      0x00000043
```
从上述结果中可以看到，正常的数据文件中，SCN值为0x0037bde8，即为10进制的3653096，Checkpoint的时间为0x3322ed7b，而需要media recovery的11号文件SCN值为0x00370826，即10进制的3606566，四个属性跟正常文件都不相同。
```
SYS@linora> select to_number('37bde8','xxxxxxxx') from dual; 
TO_NUMBER('37BDE8','XXXXXXXX')
------------------------------
                       3653096
```
再来dump一下各个offset的值(11号数据文件)：
```
--dump SCN值
BBED> set dba 11,1 offset 484 
        DBA             0x02c00001 (46137345 11,1)
        OFFSET          484
BBED> d /v count 16
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1       Offsets:  484 to  499  Dba:0x02c00001
-------------------------------------------------------
 26083700 00000000 58df2233 01000000 l &.7.....X."3....
--Checkpoint 时间
BBED> d /v dba 11,1 offset 492
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1       Offsets:  492 to  507  Dba:0x02c00001
-------------------------------------------------------
 58df2233 01000000 be000000 02000000 l X."3............
--数据文件Checkpoint次数
BBED> d /v dba 11,1 offset 140
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1       Offsets:  140 to  155  Dba:0x02c00001
-------------------------------------------------------
 44000000 c0ed2233 43000000 00000000 l D....."3C.......
--控制文件Checkpoint次数
BBED> d /v dba 11,1 offset 148
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1       Offsets:  148 to  163  Dba:0x02c00001
-------------------------------------------------------
 43000000 00000000 00000000 00000000 l C...............
```
因为此平台为Linux，属于little-endian，存储字节顺序是由左至右，因此，四个属性值和存储值对应有如下关系(0x代表16进制)：
```
00370826-->26083700
3322df58-->58df2233
00000044-->44000000
00000043-->43000000
```
由此，我们可以得出，只需要将上述四个属性值修改为1号文件相同值即可
```
--修改文件Checkpoint scn
BBED> d /v dba 1,1 offset 484 count 16
 File: /oradata/datafile/linora/system01.dbf (1)
 Block: 1       Offsets:  484 to  499  Dba:0x00400001
-------------------------------------------------------
 e8bd3700 00000000 7bed2233 01000000 l ..7.....{."3....
BBED> m /x e8bd37 dba 11,1 offset 484
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1                Offsets:  484 to  499           Dba:0x02c00001
------------------------------------------------------------------------
 e8bd3700 00000000 58df2233 01000000 
--修改文件Checkpoint 时间
BBED> d /v dba 1,1 offset 492
 File: /oradata/datafile/linora/system01.dbf (1)
 Block: 1       Offsets:  492 to  507  Dba:0x00400001
-------------------------------------------------------
 7bed2233 01000000 c3000000 61050000 l {."3........a...
BBED> m /x 7bed2233 dba 11,1 offset 492
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1                Offsets:  492 to  507           Dba:0x02c00001
------------------------------------------------------------------------
 7bed2233 01000000 be000000 02000000 
--修改文件Checkpoint次数
BBED> d /v dba 1,1 offset 140
 File: /oradata/datafile/linora/system01.dbf (1)
 Block: 1       Offsets:  140 to  155  Dba:0x00400001
-------------------------------------------------------
 87020000 b3ec2233 86020000 00000000 l ......"3........
BBED> m /x 8702 dba 11,1 offset 140
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1                Offsets:  140 to  155           Dba:0x02c00001
------------------------------------------------------------------------
 87020000 c0ed2233 43000000 00000000
--修改控制文件Checkpoint次数
BBED> d /v dba 1,1 offset 148
 File: /oradata/datafile/linora/system01.dbf (1)
 Block: 1       Offsets:  148 to  163  Dba:0x00400001
-------------------------------------------------------
 86020000 00000000 00000000 00000000 l ...............
BBED> m /x 8602 dba 11,1 offset 148
 File: /oradata/datafile/linora/test01.dbf (11)
 Block: 1                Offsets:  148 to  163           Dba:0x02c00001
------------------------------------------------------------------------
 86020000 00000000 00000000 00000000 
BBED> sum apply 
Check value for File 11, Block 1:
current = 0x90d7, required = 0x90d7
```
然而，当我修改完这四个属性值之后，启动数据库提示如下错误：
```
SYS@linora> alter database open;
alter database open
*
ERROR at line 1:
ORA-01122: database file 11 failed verification check
ORA-01110: data file 11: '/oradata/datafile/linora/test01.dbf'
ORA-01207: file is more recent than control file - old control file
```
个人觉得kcvfhcpc和kcvfhccc因数据文件而异，因为有些数据文件创建时间早，因此Checkpoint的次数就多，有些数据文件创建晚，Checkpoint的次数就少，因此，如果要修改这两个属性，必须重建控制文件。我的做法是：不修改这两个值，直接开启数据库，提示错误"ORA-01113: file 11 needs media recovery"，在rman下执行recovery，此时不会提示要apply归档日志，且数据库能正常打开了，但是，存在着数据丢失。
```
RMAN> recover datafile 11;
Starting recover at 2014-09-11 10:15:11
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=245 device type=DISK
starting media recovery
media recovery complete, elapsed time: 00:00:01
Finished recover at 2014-09-11 10:15:13
SYS@linora> alter database open; 
Database altered.
```

Reference：  
[dissassembling_the_data_block](http://www.orafaq.com/papers/dissassembling_the_data_block.pdf)
