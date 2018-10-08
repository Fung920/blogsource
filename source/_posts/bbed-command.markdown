---
layout: post
title: "BBED基本命令"
date: 2014-09-01 11:20:52
comments: false
categories: oracle
tags: bbed
keywords: bbed
description: bbed基本命令
---
前文[Linux下编译BBED](/bbed-in-linux.html)描述了如何在Linux下编译使用bbed，本文介绍bbed常用的一些命令。
<!--more-->
BBED命令帮助:
```
BBED> help all
SET DBA [ dba | file#, block# ]
SET FILENAME 'filename'
...
UNDO
HELP [ <bbed command> | ALL ]
VERIFY [ DBA | FILE | FILENAME | BLOCK ]
CORRUPT [ DBA | FILE | FILENAME | BLOCK ]
```
### 1. set命令
set主要用于定位所修改的记录所在的数据块的具体位置。在Oracle内部，数据块记录的具体位置是由RDBA和偏移量offset决定，RDBA表示修改的记录所在的数据块在数据文件的位置，offset表示修改的记录在数据块内的具体位置。
```
--查找数据dba
SYS@linora> SELECT DBMS_ROWID.ROWID_OBJECT(rowid) "OBJECT",
  2  DBMS_ROWID.ROWID_RELATIVE_FNO(rowid) "FILE",
  3  DBMS_ROWID.ROWID_BLOCK_NUMBER(rowid) "BLOCK",
  4  DBMS_ROWID.ROWID_ROW_NUMBER(rowid) "ROW"
  5  FROM hr.employees where employee_id=200;    
    OBJECT       FILE      BLOCK        ROW
---------- ---------- ---------- ----------
     84737          8      11477          2
```
bbed参数文件设置：
```
[oracle@linora:/home/oracle/bbed]$ cat parfile.txt 
blocksize=8192
listfile=filelist.txt
mode=edit
[oracle@linora:/home/oracle/bbed]$ cat filelist.txt 
1 /oradata/datafile/linora/system01.dbf 754974720
2 /oradata/datafile/linora/sysaux01.dbf 702545920
3 /oradata/datafile/linora/undotbs01.dbf 566231040
4 /oradata/datafile/linora/users01.dbf 36700160
5 /oradata/datafile/linora/fung01.dbf 246415360
6 /oradata/datafile/linora/undotbs02.dbf 1048576
7 /oradata/datafile/linora/fung02.dbf 10485760
8 /oradata/datafile/linora/perf01.dbf 314572800
9 /oradata/datafile/linora/demotsdata.dbf 104857600
10 /oradata/datafile/linora/demotsidx.dbf 104857600
11 /oradata/datafile/linora/test01.dbf 1048576
```
#### set dba
使用DBA定位数据块位置，默认offset为0，也可以指定offset。
```
BBED> set dba 8,11477
        DBA             0x02002cd5 (33565909 8,11477)
BBED> d
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets:    0 to  511           Dba:0x02002cd5
--默认offset为0
```
#### set filename
以数据文件名指定当前修改的数据文件，需要绝对路径
```
BBED> set filename '/oradata/datafile/linora/perf01.dbf'
        FILENAME        /oradata/datafile/linora/perf01.dbf
```
#### set file
以file_id指定当前修改的数据文件
```
BBED> set file 8
        FILE#           8
```
#### set block
指定当前修改的块号，在设定块号之前，需要先指定数据文件，可使用绝对块号，或者使用'+'，'-'号指定离当前块号距离的目标块号
```
BBED> set block 11476
        BLOCK#          11476
BBED> set block +1
        BLOCK#          11477
```
#### set offset
指定当前块内的起始偏移量，用法跟set block类似，也可用'+'，'-'号来表示跟当前offset的相对offset
```
BBED> set offset 10
        OFFSET          10
BBED> d
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets:   10 to  521           Dba:0x02002cd5
--offset为10
BBED> set offset +10
        OFFSET          20
BBED> d
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets:   20 to  531           Dba:0x02002cd5
--offset为20
```
#### set blocksize
设置当前数据文件的块大小，必须是当前数据文件的块大小，否则报错
```
BBED> set blocksize 8192
        BLOCKSIZE       8192
BBED> set blocksize 16384
BBED-00307: incorrect blocksize (8192) or truncated file
```
#### set listfile
设置使用的listfile文件，listfile文件包含bbed所要编辑的数据文件列表 
```
BBED> set listfile 'filelist.txt'
        LISTFILE        filelist.txt
```
#### set count
设置dump命令显示的对应数据块的字节数，默认为512个字节。如果需要看到一个8k数据块的整个块内容，可以设置count为8192或者更大 
```
BBED> set dba 8,11477
        DBA             0x02002cd5 (33565909 8,11477)
BBED> d 
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets:    0 to  511           Dba:0x02002cd5
--默认dump显示为0~511即512个字节
BBED> set count 8192
        COUNT           8192
BBED> d
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets:    0 to 8191           Dba:0x02002cd5
--设置count为8k后，显示字节为0~8191即8192个字节
```
#### set ibase
设置使用set block,set file,set offset使用的进制:Dec-十进制 Hex-十六进制 Oct-八进制，默认为10进制
```
BBED> show ibase
        IBASE           Dec
```
#### set obase
此命令未知。
#### set mode
设置bbed模式是编辑(edit)还是浏览(browse)，浏览模式无法对数据块进行修改
```
BBED> show mode
        MODE            Edit
BBED> set mode browse
        MODE            Browse
BBED> show mode
        MODE            Browse
```
### 2. show命令
显示当前bbed的配置
```
BBED> set dba 8, 11476
        DBA             0x02002cd4 (33565908 8,11476)
BBED> show
        FILE#           8
        BLOCK#          11476
        OFFSET          0
        DBA             0x02002cd4 (33565908 8,11476)
        FILENAME        /oradata/datafile/linora/perf01.dbf
        BIFILE          bifile.bbd
        LISTFILE        filelist.txt
        BLOCKSIZE       8192
        MODE            Edit
        EDIT            Unrecoverable
        IBASE           Dec
        OBASE           Dec
        WIDTH           80
        COUNT           512
        LOGFILE         log.bbd
        SPOOL           No
```

### 3. info命令
显示当前的listfile内容 
```
BBED> info
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /oradata/datafile/linora/system01.dbf                            92160
     2  /oradata/datafile/linora/sysaux01.dbf                            85760
     3  /oradata/datafile/linora/undotbs01.dbf                           69120
     4  /oradata/datafile/linora/users01.dbf                              4480
     5  /oradata/datafile/linora/fung01.dbf                              30080
     6  /oradata/datafile/linora/undotbs02.dbf                             128
     7  /oradata/datafile/linora/fung02.dbf                               1280
     8  /oradata/datafile/linora/perf01.dbf                              38400
     9  /oradata/datafile/linora/demotsdata.dbf                          12800
    10  /oradata/datafile/linora/demotsidx.dbf                           12800
    11  /oradata/datafile/linora/test01.dbf                                128
```
### 4. map命令
map显示数据块结构，如果加上<code>/v</code>则表示显示数据块结构的同时显示结构体(Oracle 定义的Structure)的每个字段。
```
--map不加dba表示显示当前块结构及字段，加上dba表示显示指定的块结构和字段
BBED> map /v dba 8,11476
BBED> map /v dba 8,11476
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11476                                 Dba:0x02002cd4
------------------------------------------------------------
 KTB Data Block (Table/Cluster)

 struct kcbh, 20 bytes                      @0       
    ub1 type_kcbh                           @0       
    ub1 frmt_kcbh                           @1       
    ub1 spare1_kcbh                         @2       
    ub1 spare2_kcbh                         @3       
    ub4 rdba_kcbh                           @4       
    ub4 bas_kcbh                            @8       
    ub2 wrp_kcbh                            @12      
    ub1 seq_kcbh                            @14      
    ub1 flg_kcbh                            @15      
    ub2 chkval_kcbh                         @16      
    ub2 spare3_kcbh                         @18      

 struct ktbbh, 72 bytes                     @20      
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[2], 48 bytes            @44      

 struct kdbh, 14 bytes                      @100     
    ub1 kdbhflag                            @100     
    sb1 kdbhntab                            @101     
    sb2 kdbhnrow                            @102     
    sb2 kdbhfrre                            @104     
    sb2 kdbhfsbo                            @106     
    sb2 kdbhfseo                            @108     
    sb2 kdbhavsp                            @110     
    sb2 kdbhtosp                            @112     

 struct kdbt[1], 4 bytes                    @114     
    sb2 kdbtoffs                            @114     
    sb2 kdbtnrow                            @116     

 sb2 kdbr[98]                               @118     

 ub1 freespace[686]                         @314     

 ub1 rowdata[7188]                          @1000    

 ub4 tailchk                                @8188    
```
### 5. dump命令
dump命令用于查看指定block、指定offset的数据块内记录的内容，这些内容以16进制记载。相当于对当前block进行strings操作。此命令可缩写为d。
```
BBED> set dba 8, 11476 offset 1024 count 20
        DBA             0x02002cd4 (33565908 8,11476)
        OFFSET          1024
        COUNT           20
BBED> d /v
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11476   Offsets: 1024 to 1043  Dba:0x02002cd4
-------------------------------------------------------
 0c353135 2e313233 2e343536 37077867 l .515.123.4567.xg
 06110101                            l ....
<16 bytes per line>
--或者
BBED> d /v dba 8,11476 offset 1024 count 50
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11476   Offsets: 1024 to 1073  Dba:0x02002cd4
-------------------------------------------------------
 0c353135 2e313233 2e343536 37077867 l .515.123.4567.xg
 06110101 01074144 5f505245 5304c304 l ......AD_PRES...
 2e3dffff 02c15b2c 000b02c2 02065374 l .=....[,......St
 6576                                l ev
```
### 6. print命令
显示数据块中offset位置的块结构。
```
--打印指定dba offset，下面是返回kcbh及数据块头(Data Block Header)
BBED> p dba 8,11476 offset 0
kcbh.type_kcbh
--------------
ub1 type_kcbh                               @0        0x06
```
打印指定数据结构，下面的例子中，分三列，第三列是具体值
```
BBED> p kcbh
struct kcbh, 20 bytes                       @0       
   ub1 type_kcbh                            @0        0x06
   ub1 frmt_kcbh                            @1        0xa2
   ub1 spare1_kcbh                          @2        0x00
   ub1 spare2_kcbh                          @3        0x00
   ub4 rdba_kcbh                            @4        0x02002cd4
   ub4 bas_kcbh                             @8        0x00316808
   ub2 wrp_kcbh                             @12       0x0000
   ub1 seq_kcbh                             @14       0x01
   ub1 flg_kcbh                             @15       0x06 (KCBHFDLC, KCBHFCKV)
   ub2 chkval_kcbh                          @16       0x8920
   ub2 spare3_kcbh                          @18       0x0000
```
跟dump结果对应下，可以看到，data block Header，dump存储的数据跟print的数据存在一定的关系。
```
BBED> set count 20
        COUNT           20
BBED> d
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11476            Offsets:    0 to   19           Dba:0x02002cd4
------------------------------------------------------------------------
 06a20000 d42c0002 08683100 00000106 20890000 
```
print还可以通过kdbr[n]来定位到当前块的第n行记录，数据块存储记录是有0行开始。
```
--定位数据块内第三行记录
BBED> p dba 8,11477 *kdbr[2]
rowdata[410]
------------
ub1 rowdata[410]                            @7976     0x2c
```
显示当前偏移量的值为2c，和dump出来的第一个字节相同，即普通数据块中行记录所在的行头的标识。
```
BBED> d /v dba 8,11477 offset 7976 count 512
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477   Offsets: 7976 to 8191  Dba:0x02002cd5
-------------------------------------------------------
 2c010b02 c203084a 656e6e69 66657206 l ,......Jennifer.
 5768616c 656e074a 5748414c 454e0c35 l Whalen.JWHALEN.5
 31352e31 32332e34 34343407 78670911 l 15.123.4444.xg..
 01010107 41445f41 53535402 c22dff03 l ....AD_ASST..-..
 c2020202 c10b2c01 0b03c202 6407446f l ......,.....d.Do
 75676c61 73054772 616e7406 44475241 l uglas.Grant.DGRA
 4e540c36 35302e35 30372e39 38343407 l NT.650.507.9844.
 786c010d 01010108 53485f43 4c45524b l xl......SH_CLERK
 02c21bff 03c20219 02c1332c 010b03c2 l ..........3,....
 02630644 6f6e616c 64084f43 6f6e6e65 l .c.Donald.OConne
 6c6c0844 4f434f4e 4e454c0c 3635302e l ll.DOCONNEL.650.
 3530372e 39383333 07786b06 15010101 l 507.9833.xk.....
 0853485f 434c4552 4b02c21b ff03c202 l .SH_CLERK.......
 1902c133 010695d7                   l ...3....
 <32 bytes per line>
```
print的参数如下：

parameters|format
----|----
/x|十六进制
/d|带符号十进制
/u|无符号十进制
/o|八进制
/c|字符
/n|number
/t|Oracle时间类型
/i|Oracle rowid

```
BBED> p /c rowdata 
ub1 rowdata[0]                              @7566    ,
ub1 rowdata[1]                              @7567    .
ub1 rowdata[2]                              @7568    .
...
ub1 rowdata[603]                            @8169    S
ub1 rowdata[604]                            @8170    H
ub1 rowdata[605]                            @8171    _
ub1 rowdata[606]                            @8172    C
ub1 rowdata[607]                            @8173    L
ub1 rowdata[608]                            @8174    E
ub1 rowdata[609]                            @8175    R
ub1 rowdata[610]                            @8176    K
...
```
### 7. examine命令
此命令用于将当前数据块的行记录从16进制翻译成文本,其后可接参数/r，表示read，且/r后可接一堆参数，其中n表示number，c表示character，即varchar或者varchar2类型的字符串，t表示日期类型。
```
--table bigtable字段为ccnc类型
FUNG@linora> desc bigtable
 Name                           Null?    Type
 ------------------------------ -------- -------------------
 ID                                      VARCHAR2(10)
 INC_DATETIME                            VARCHAR2(19)
 RANDOM_ID                               NUMBER
 RANDOM_STRING                           VARCHAR2(4000)
--查找某条记录
FUNG@linora> select * from bigtable where id='101401999';
ID         INC_DATETIME         RANDOM_ID RANDOM_STRING
---------- ------------------- ---------- ----------------------------
101401999  2014-07-16 11:01:37         10 EST7QUM940KAEOZSV2ZF
--查找某条记录的dba
FUNG@linora> SELECT DBMS_ROWID.ROWID_OBJECT(rowid) "OBJECT",
  2  DBMS_ROWID.ROWID_RELATIVE_FNO(rowid) "FILE",
  3  DBMS_ROWID.ROWID_BLOCK_NUMBER(rowid) "BLOCK",
  4  DBMS_ROWID.ROWID_ROW_NUMBER(rowid) "ROW"
  5  FROM fung.bigtable where id='101401999'; 

    OBJECT       FILE      BLOCK        ROW
---------- ---------- ---------- ----------
     85368          5        243          0
```
使用bbed的examine命令打印当前行记录。
```
BBED> set dba 5,243
        DBA             0x014000f3 (20971763 5,243)
BBED> p *kdbr[0]
rowdata[7703]
-------------
ub1 rowdata[7703]                           @8131     0x2c
BBED> x /rccnc
rowdata[7703]                               @8131    
-------------
flag@8131: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8132: 0x00
cols@8133:    4

col    0[9] @8134: 101401999
col   1[19] @8144: 2014-07-16 11:01:37
col    2[2] @8164: 10 
col   3[20] @8167: EST7QUM940KAEOZSV2ZF
```
可以看到bbed出来的结果记录和select语句一致，如果/r后的c和n调换位置，则会出现其他情况：
```
BBED> x /rnnc
rowdata[7703]                               @8131    
-------------
flag@8131: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8132: 0x00
cols@8133:    4

col    0[9] @8134: ######################################### 
col   1[19] @8144: ######################################### 
col    2[2] @8164: ..
col   3[20] @8167: EST7QUM940KAEOZSV2ZF
```
由此可得知，使用x命令进行read的时候，要和表定义的数据类型一致。
### 8. find命令
find用于查找指定字符串在指定数据块的具体位置，可简写为f，查找出符合条件后，再输入f会显示下一条符合条件的记录。find命令参数如下表所示

parameter|datatype
----|----
/x|十六进制
/d|十进制
/u|无符号十进制
/o|八进制
/c|字符

```
BBED> set dba 8,11477
        DBA             0x02002cd5 (33565909 8,11477)
--从当前offset查找
BBED> find /c Jen CURR 
BBED-00212: search string not found
--从offset=0开始查找
BBED> find /c Jen TOP
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets: 7983 to 8191           Dba:0x02002cd5
------------------------------------------------------------------------
 4a656e6e 69666572 06576861 6c656e07 4a574841 4c454e0c 3531352e 3132332e 
 34343434 07786709 11010101 0741445f 41535354 02c22dff 03c20202 02c10b2c 
 010b03c2 02640744 6f75676c 61730547 72616e74 06444752 414e540c 3635302e 
 3530372e 39383434 07786c01 0d010101 0853485f 434c4552 4b02c21b ff03c202 
 1902c133 2c010b03 c2026306 446f6e61 6c64084f 436f6e6e 656c6c08 444f434f 
 4e4e454c 0c363530 2e353037 2e393833 3307786b 06150101 01085348 5f434c45 
 524b02c2 1bff03c2 021902c1 33010695 d7 
--此时的offset被设置为匹配搜索的第一个位置
BBED> show offset
        OFFSET          7983
--验证当前offset是否为Jen
BBED> p /c 
rowdata[417]
------------
ub1 rowdata[417]                            @7983    J
BBED> p /c offset +1
rowdata[418]
------------
ub1 rowdata[418]                            @7984    e
BBED> p /c offset +1
rowdata[419]
------------
ub1 rowdata[419]                            @7985    n
```
### 9. copy命令
copy用于复制一个数据块到另一个地方，如：
```
BBED> copy dba 8,11476 to dba 11,10240
```
### 10. modify命令
modify为修改块内数据的命令，其命令指定的数据格式跟find相同，可简写为m。如修改dba 8,block 11477的Jen为JAN：
```
--查找Jen所在offset
BBED> f /c Jen TOP
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets: 7983 to 8191           Dba:0x02002cd5
------------------------------------------------------------------------
 4a656e6e 69666572 06576861 6c656e07 4a574841 4c454e0c 3531352e 3132332e 
 34343434 07786709 11010101 0741445f 41535354 02c22dff 03c20202 02c10b2c 
 010b03c2 02640744 6f75676c 61730547 72616e74 06444752 414e540c 3635302e 
 3530372e 39383434 07786c01 0d010101 0853485f 434c4552 4b02c21b ff03c202 
 1902c133 2c010b03 c2026306 446f6e61 6c64084f 436f6e6e 656c6c08 444f434f 
 4e4e454c 0c363530 2e353037 2e393833 3307786b 06150101 01085348 5f434c45 
 524b02c2 1bff03c2 021902c1 33010695 d7 
--验证offset是否以Jen开头
BBED> d /v
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477   Offsets: 7983 to 8191  Dba:0x02002cd5
-------------------------------------------------------
 4a656e6e 69666572 06576861 6c656e07 l Jennifer.Whalen.
 4a574841 4c454e0c 3531352e 3132332e l JWHALEN.515.123.
 34343434 07786709 11010101 0741445f l 4444.xg......AD_
 41535354 02c22dff 03c20202 02c10b2c l ASST..-........,
 010b03c2 02640744 6f75676c 61730547 l .....d.Douglas.G
 72616e74 06444752 414e540c 3635302e l rant.DGRANT.650.
 3530372e 39383434 07786c01 0d010101 l 507.9844.xl.....
 0853485f 434c4552 4b02c21b ff03c202 l .SH_CLERK.......
 1902c133 2c010b03 c2026306 446f6e61 l ...3,.....c.Dona
 6c64084f 436f6e6e 656c6c08 444f434f l ld.OConnell.DOCO
 4e4e454c 0c363530 2e353037 2e393833 l NNEL.650.507.983
 3307786b 06150101 01085348 5f434c45 l 3.xk......SH_CLE
 524b02c2 1bff03c2 021902c1 33010695 l RK..........3...
 d7                                  l .
--以字符格式进行修改
BBED> m /c JAN dba 8,11477 offset 7983
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477            Offsets: 7983 to 8191           Dba:0x02002cd5
------------------------------------------------------------------------
 4a414e6e 69666572 06576861 6c656e07 4a574841 4c454e0c 3531352e 3132332e 
 34343434 07786709 11010101 0741445f 41535354 02c22dff 03c20202 02c10b2c 
 010b03c2 02640744 6f75676c 61730547 72616e74 06444752 414e540c 3635302e 
 3530372e 39383434 07786c01 0d010101 0853485f 434c4552 4b02c21b ff03c202 
 1902c133 2c010b03 c2026306 446f6e61 6c64084f 436f6e6e 656c6c08 444f434f 
 4e4e454c 0c363530 2e353037 2e393833 3307786b 06150101 01085348 5f434c45 
 524b02c2 1bff03c2 021902c1 33010695 d7 
--验证修改结果
BBED> d /v
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477   Offsets: 7983 to 8191  Dba:0x02002cd5
-------------------------------------------------------
 4a414e6e 69666572 06576861 6c656e07 l JANnifer.Whalen.
 4a574841 4c454e0c 3531352e 3132332e l JWHALEN.515.123.
 34343434 07786709 11010101 0741445f l 4444.xg......AD_
 41535354 02c22dff 03c20202 02c10b2c l ASST..-........,
 010b03c2 02640744 6f75676c 61730547 l .....d.Douglas.G
 72616e74 06444752 414e540c 3635302e l rant.DGRANT.650.
 3530372e 39383434 07786c01 0d010101 l 507.9844.xl.....
 0853485f 434c4552 4b02c21b ff03c202 l .SH_CLERK.......
 1902c133 2c010b03 c2026306 446f6e61 l ...3,.....c.Dona
 6c64084f 436f6e6e 656c6c08 444f434f l ld.OConnell.DOCO
 4e4e454c 0c363530 2e353037 2e393833 l NNEL.650.507.983
 3307786b 06150101 01085348 5f434c45 l 3.xk......SH_CLE
 524b02c2 1bff03c2 021902c1 33010695 l RK..........3...
 d7                                  l .
```
注意，在以前的bbed版本中进行修改，会以交互模式提醒是否确定要修改，但11g以后已经没有了这个提示。
### 11. sum命令
sum命令用来检测和设置block的Checksum。
```
--验证数据块Checksum
BBED> sum dba 8,11477
Check value for File 8, Block 11477:
current = 0x62bf, required = 0x429b
--更新数据块Checksum
BBED> sum dba 8,11477 apply
Check value for File 8, Block 11477:
current = 0x429b, required = 0x429b
```
更新完毕后，可以刷新缓冲池，在数据库内查看修改结果。
```
SYS@linora> alter system flush buffer_cache;
System altered.

SYS@linora> SELECT DBMS_ROWID.ROWID_RELATIVE_FNO(rowid) "FILE",
  2  DBMS_ROWID.ROWID_BLOCK_NUMBER(rowid) "BLOCK",first_name
  3  FROM hr.employees where employee_id=200;
      FILE      BLOCK FIRST_NAME
---------- ---------- --------------------
         8      11477 JANnifer
```

Reference：  
[dissassembling_the_data_block](http://www.orafaq.com/papers/dissassembling_the_data_block.pdf)
