---
layout: post
title: "重做记录"
date: 2014-08-24 09:58:33
comments: false
categories: oracle
tags: internal
keywords: redo record
description: 重做记录详解
---
大师Jonathan Lewis在《Oracle Core》提到Oracle的一个重要特性，就是在Oracle 6中首次提出的change vector，即改变向量。改变向量是描述数据块变化的机制，是redo和undo的核心。
<!--more-->
关于数据改变的方法，大师在《Oracle Core》描述如下：
<li>创建一个重做改变向量，描述如何往undo块插入一条undo记录</li>
<li>创建一个重做改变向量，描述数据块的改变</li>
<li>合并这两个重做记录为一条日志记录，并写到重做日志缓冲区</li>
<li>向undo块插入undo记录</li>
<li>改变数据块中的数据</li>
在理解重做记录和改变向量之前，需要理解SCN、数据块版本号和RBA等概念。
### 1. 基本概念
#### 1.1 SCN 
SCN是Oracle的内部时钟机制，SCN由两部分共6字节组成，4字节为base(底值)，2字节为Wrap(进位值)。当前的base随时间的流逝和变更操作的执行而递增，在用尽之后开始循环，此时wrap递增。
```
--查询SCN的方法
SYS@linora> select current_scn from v$database
  2  union all
  3  select dbms_flashback.get_system_change_number from dual;

CURRENT_SCN
-----------
    3107373
    3107373
```
如果之前数据库是干净关闭的，mount状态下查看控制文件和数据文件头部信息的SCN，可以发现这两者的值是一样的：
```
--dump控制文件的scn
SYS@linora> alter session set events 'immediate trace name controlf level 8';
Session altered.
--Trace 文件内容
Database checkpoint: Thread=1 scn: 0x0000.002ded1f--Database checkpoint scn
...
Controlfile Checkpointed at scn:  0x0000.002de9f1 08/22/2014 14:36:37 -- controlfile checkpoint scn
...
***************************************************************************
DATA FILE RECORDS
***************************************************************************
 (size = 520, compat size = 520, section max = 100, section in-use = 10,
  last-recid= 161, old-recno = 0, last-recno = 0)
 (extent = 1, blkno = 11, numrecs = 100)
DATA FILE # 1:
  name # 4: /oradata/datafile/linora/system01.dbf
Checkpoint cnt:539 scn: 0x0000.002ded1f 08/22/2014 14:46:09 -- datafile checkpoint scn in controflile 
...
DATA FILE # 2:
  name # 5: /oradata/datafile/linora/sysaux01.dbf
creation size=76800 block size=8192 status=0xe head=5 tail=5 dup=1
 tablespace 1, index=2 krfil=2 prev_file=0
 unrecoverable scn: 0x0000.00000000 01/01/1988 00:00:00
 Checkpoint cnt:534 scn: 0x0000.002ded1f 08/22/2014 14:46:09
--尝试在mount状态下dump数据文件和数据文件头，发现和控制文件dump的scn是一致的
SYS@linora> alter session set events 'immediate trace name file_hdrs level 8';
Session altered.
DATA FILE # 1:
  name # 4: /oradata/datafile/linora/system01.dbf
creation size=89600 block size=8192 status=0xe head=4 tail=4 dup=1
 tablespace 0, index=1 krfil=1 prev_file=0
 unrecoverable scn: 0x0000.00000000 01/01/1988 00:00:00
 Checkpoint cnt:539 scn: 0x0000.002ded1f 08/22/2014 14:46:09
 Stop scn: 0x0000.002ded1f 08/22/2014 14:46:09
 ...
DATA FILE # 2:
  name # 5: /oradata/datafile/linora/sysaux01.dbf
creation size=76800 block size=8192 status=0xe head=5 tail=5 dup=1
 tablespace 1, index=2 krfil=2 prev_file=0
 unrecoverable scn: 0x0000.00000000 01/01/1988 00:00:00
 Checkpoint cnt:534 scn: 0x0000.002ded1f 08/22/2014 14:46:09
 Stop scn: 0x0000.002ded1f 08/22/2014 14:46:09
```
SCN是以scn: 0x0000.002ded1f存储在dump文件中，可以通过函数转换成16进制：
```
SYS@linora> select to_number('2ded1f','xxxxxx') from dual;
TO_NUMBER('2DED1F','XXXXXX')
----------------------------
                     3009823
```
#### 1.2 数据块版本
当同一个SCN包含多条重做记录时候，说明有一个以上的操作分配到了同样的SCN，此时，引入一个SUBSCN，如'SCN: 0x0000.002de89d SUBSCN:  1'，当修改完成后，SCN和SUBSCN会被保存在被修改的数据块的头部，占用7字节，但此时SUBSCN变为SEQ，SCN+SEQ就是数据块版本号：
```
SYS@linora> ALTER SYSTEM DUMP LOGFILE '/oradata/datafile/linora/redo02.log';
System altered.
--redo record头部信息
REDO RECORD - Thread:1 RBA: 0x000097.00000002.0010 LEN: 0x026c VLD: 0x05
SCN: 0x0000.002de89d SUBSCN:  1 08/22/2014 14:22:02 --修改前版本号
```
#### 1.3 RBA
RBA全称为Redo Byte Address，其作用类似rowid对于表的作用。即重做日志中的物理地址，RBA由四部分组成：日志线程号、日志序列号、日志文件块号和日志文件块字节偏移量，共10字节。
```
SYS@linora> ALTER SYSTEM DUMP LOGFILE '/oradata/datafile/linora/redo02.log';
System altered.
SYS@linora> select value from v$diag_info where name='Default Trace File';
--Trace文件部分信息
DUMP OF REDO FROM FILE '/oradata/datafile/linora/redo02.log'
REDO RECORD - Thread:1 RBA: 0x00009a.00000002.0010 LEN: 0x0080 VLD: 0x06
SCN: 0x0000.002f0ea5 SUBSCN:  1 08/23/2014 22:56:28
```
可以看到，在dump文件中，RBA以RBA: 0x00009a.00000002.0010的形式存在，其含义为Thread 1，log sequence=154的redo log的第2个块中的第16字节处，对于16进制转换，参照如下方法，文件结尾后面总结了一些比较常用的进制转换方法。
```
SYS@linora> select to_number('9a','xxxxxxx') from dual;
TO_NUMBER('9A','XXXXXXX')
-------------------------
                      154
```
### 2. 重做记录
本例以修改hr.empployees中员工号为100的薪资(修改后薪水为原来的120%)为例，来简单探索重做记录和改变向量。
```
--先查看当前记录所在行的物理地址和原来的薪水
SYS@linora> select rowid,salary,first_name,last_name
  2  from hr.employees where employee_id=100;
ROWID                  SALARY FIRST_NAME           LAST_NAME
------------------ ---------- -------------------- -------------------------
AAAUsBAAIAAACzUAAA      24000 Steven               King
```
rowid以Base64的形式出现，该结构前6位代表segment_id—AAAUsB(84737)，随后三位代表数据文件号—AAI(8)，接下来的6位代表数据块编号—AAACzU(11476)，最后三位代表行号—AAA(0)。 因此，在上述结果中，emp id=100的员工其数据存在的物理位置为：存储在第8号文件的第11476块上第0行(关于ROWID的说明请参照[Oracle ROWID学习](/what-is-rowid.html))。使用dbms_rowid函数查看：
```
SYS@linora> SELECT DBMS_ROWID.ROWID_OBJECT(rowid) "OBJECT",
  2  DBMS_ROWID.ROWID_RELATIVE_FNO(rowid) "FILE",
  3  DBMS_ROWID.ROWID_BLOCK_NUMBER(rowid) "BLOCK",
  4  DBMS_ROWID.ROWID_ROW_NUMBER(rowid) "ROW"
  5  FROM hr.employees where employee_id=100;
    OBJECT       FILE      BLOCK        ROW
---------- ---------- ---------- ----------
     84737          8      11476          0
```
由于redo log包含的范围比较广，因此在dump日志文件的时候可基于scn值去dump相关时间段的内容(其他方式的dump，如基于DBA，请参照文章结尾)：
```
SYS@linora> set serverout on
SYS@linora> declare
  2       SCN1 number;
  3       SCN2 number;
  4      begin
  5          select current_scn into SCN1 from v$database;
  6          dbms_output.put_line('Before Updata SCN: '||SCN1);
  7          update hr.employees set salary=salary*1.2 where employee_id=100;
  8         commit;
  9         select current_scn into SCN2 from v$database;
 10         dbms_output.put_line('After Update SCN: '||SCN2);
 11  end;
 12  /
Before Updata SCN: 3109548
After Update SCN: 3109551
PL/SQL procedure successfully completed.
```
查找当前日志文件所在路径：
```
SYS@linora> col member for a50
SYS@linora> select member,sequence# from v$logfile a,v$log b
  2  where b.group#=a.group#
  3  and b.status='CURRENT';
MEMBER                                              SEQUENCE#
-------------------------------------------------- ----------
/oradata/datafile/linora/redo03.log                       155
```
dump相关日志文件：
```
SYS@linora> alter system dump logfile '/oradata/datafile/linora/redo03.log' scn min 3109548 scn max 3109551;
System altered.
```
#### 2.1 头部信息
```
REDO RECORD - Thread:1 RBA: 0x00009b.00002c33.0010 LEN: 0x0278 VLD: 0x05
SCN: 0x0000.002f72ac SUBSCN:  1 08/24/2014 17:14:38
(LWN RBA: 0x00009b.00002c33.0010 LEN: 0002 NST: 0001 SCN: 0x0000.002f72ab)
```
重做记录头部信息包含了以下信息：RBA，数据块版本号等。以上信息表明重做记录的头部信息记录在Thread 1，log sequence#为155上(和之前查询的结果一致)的第11315块上的第16字节处。当前数据块版本号：SCN: 0x0000.002f72ac SUBSCN:  1，转换如下：
```
SYS@linora> select to_number('9b','xxxxxxx') from dual;
TO_NUMBER('9B','XXXXXXX')
-------------------------
                      155
SYS@linora> select to_number('2c33','xxxxxxx') from dual;
TO_NUMBER('2C33','XXXXXXX')
---------------------------
                      11315
SYS@linora> select to_number('10','xxxxxx') from dual;
TO_NUMBER('10','XXXXXX')
------------------------
                      16
```
#### 2.2 CHANGE # 1
```
CHANGE # 1 TYP:0 CLS:23 AFN:3 DBA:0x00c000b0 OBJ:4294967295 SCN:0x0000.002f724b SEQ:1 OP:5.2 ENC:0 RBL:0
ktudh redo: slt: 0x0004 sqn: 0x00000530 flg: 0x000a siz: 184 fbi: 0
            uba: 0x00c0cc3e.00f6.01    pxid:  0x0000.000.00000000
```
第一个改变向量是记录了如何对3号文件的176块进行修改，该数据块是undo segment头部，地址由AFN绝对文件号(对应v$datafile.file#)和DBA(Data Block Address)表示。当前数据块版本号为：SCN:0x0000.002f724b SEQ:1，修改数据后此数据块的版本号为:SCN: 0x0000.002f72ac SUBSCN:  1 (来自重做记录头部信息)。其中DBA包括文件编号和数据块编号，长度为4字节，使用dbms_utility可以进行转换：
```
SYS@linora> select to_number('c000b0','xxxxxxx') from dual;
TO_NUMBER('C000B0','XXXXXXX')
-----------------------------
                     12583088
SYS@linora> select dbms_utility.data_block_address_file(12583088) as rfile,
dbms_utility.data_block_address_block(12583088) as block from dual;
     RFILE      BLOCK
---------- ----------
         3        176
```
通过AFN和DBA查找对象：
```
SYS@linora> col tablespace_name for a20 
SYS@linora> col segment_type for a10 
SYS@linora> col segment_name for a20 
SYS@linora> col owner for a8 
SYS@linora> SELECT tablespace_name, segment_type, owner, segment_name 
  2  FROM dba_extents 
  3  WHERE file_id = &fileid 
  4  and &blockid between block_id AND block_id + blocks - 1;
Enter value for fileid: 3
old   3: WHERE file_id = &fileid
new   3: WHERE file_id = 3
Enter value for blockid: 176
old   4: and &blockid between block_id AND block_id + blocks - 1
new   4: and 176 between block_id AND block_id + blocks - 1

TABLESPACE_NAME      SEGMENT_TY OWNER    SEGMENT_NAME
-------------------- ---------- -------- --------------------
UNDOTBS1             TYPE2 UNDO SYS      _SYSSMU4_3538150892$
```
很明显，当前矢量修改的是undo segment '_SYSSMU4_3538150892$'。修改undo segment的目的是创建事务表。这是因为update命令发起了事务，Oracle必须为每个事务分配undo segment，在undo segment中创建一张事务表，该表用来保存事务产生的旧数据，用于支持事务的回滚和查询的一致性读。一个事务产生的所有旧数据都存放在同一个事务表中，一个undo segment又可以包含多个事务表。
#### 2.3 CHANGE # 2
```
CHANGE # 2 TYP:1 CLS:24 AFN:3 DBA:0x00c0cc3e OBJ:4294967295 SCN:0x0000.002f72ac SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 184 spc: 0 flg: 0x000a seq: 0x00f6 rec: 0x01
            xid:  0x0004.004.00000530  
ktubl redo: slt: 4 rci: 0 opc: 11.1 [objn: 84737 objd: 84737 tsn: 8]
Undo type:  Regular undo        Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
             0x00000000  prev ctl uba: 0x00c0cc3d.00f6.22 
prev ctl max cmt scn:  0x0000.002f6e01  prev tx cmt scn:  0x0000.002f6e0a 
txn start scn:  0xffff.ffffffff  logon user: 0  prev brb: 12635193  prev bcl: 0 BuExt idx: 0 flg2: 0
KDO undo record:
KTB Redo 
op: 0x04  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: L  itl: xid:  0x000a.020.00000513 uba: 0x00c003fe.00d2.01
                      flg: C---    lkc:  0     scn: 0x0000.002f0291
KDO Op code: URP row dependencies Disabled --Operation Code为Update
  xtype: XA flags: 0x00000000  bdba: 0x02002cd4  hdba: 0x02002cd2 --bdba:正在更新的块的地址 hdba:该对象段头块地址
itli: 2  ispac: 0  maxfr: 4858 --itli：事务槽，即第二个事务槽
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 0 
ncol: 11 nnew: 1 size: 0 	--size：表示存储数据是否改变，如果原来是三位，后面增加了一位，则size：1
col  7: [ 3]  c3 03 29 --update前的数据
```
第一个矢量负责创建事务表，第二个则负责在事务表中创建具体的撤销数据，以上信息记录了如何对AFN=3，DBA:0x00c0cc3e(52286号)的数据块进行修改。在tabn中，slot：0(0x0)表示数据块中的第一行被update命令修改，col 7: [ 3] c3 03 29表示改行的第8个字段，在update前是16进制c30329，长度3字节，可以通过dump查看：
```
SYS@linora> select dump(24000,16) from dual;
DUMP(24000,16)
--------------------
Typ=2 Len=3: c3,3,29
```
#### 2.4 CHANGE # 3
前两个变更矢量只是表明undo record该如何生成，尚未提及100号员工所在的数据块—8号文件11476块如何修改，这是CHANGE # 3的任务。
```
CHANGE # 3 TYP:2 CLS:1 AFN:8 DBA:0x02002cd4 OBJ:84737 SCN:0x0000.002f7210 SEQ:1 OP:11.5 ENC:0 RBL:0
KTB Redo 
op: 0x11  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: F  xid:  0x0004.004.00000530    uba: 0x00c0cc3e.00f6.01
Block cleanout record, scn:  0x0000.002f72ac ver: 0x01 opt: 0x02, entries follow...
  itli: 1  flg: 2  scn: 0x0000.002f7210
KDO Op code: URP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x02002cd4  hdba: 0x02002cd2
itli: 2  ispac: 0  maxfr: 4858
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 2 ckix: 0
ncol: 11 nnew: 1 size: 0
col  7: [ 3]  c3 03 59  --修改后的数据
```
第一行记录了AFN和DBA，tabn记录了slot(行)及最后以后记录了col(字段)，因此，这个矢量才是update的真正目的。将8号文件11476块的第一行(slot 0)的第八个字段(col 7)修改为c30359，这个16进制的转换涉及到Oracle存储Number类型的内部原理，可参照老盖的Blog:[How Oracle Store Number internal](http://www.eygle.com/archives/2005/12/how_oracle_stor.html)，根据老盖的介绍作一个简要的说明： 由于[ 3]是表示以三位存储这个16进制，因此，需要分开c3-03-59分别转换成10进制：
```
SYS@linora> select to_number('c3','xxxxxx') from dual;
TO_NUMBER('C3','XXXXXX')
------------------------
                     195
SYS@linora> select to_number('59','xxxxxx') from dual;
TO_NUMBER('59','XXXXXX')
------------------------
                      89
```
因此，实际存储则为十进制:195,3,89，按照老盖的说明： 指数：195-193=2，为正数，数字1：3-1 = 21002-0 20000 ，数字2: 89 -1 =881001-0 8800 。因此，这个Number类型的值为20000+8800=28800跟我们预期的结果一致！使用dump函数dump一下28800的16进制和10进制：
```
--16进制
SYS@linora> select dump(28800,16) from dual;
DUMP(28800,16)
--------------------
Typ=2 Len=3: c3,3,59
--10进制
SYS@linora> select dump(28800,10) from dual;
DUMP(28800,10)
---------------------
Typ=2 Len=3: 195,3,89
```
用其他函数验证(dump逆函数)：
```
--16进制转10进制
SYS@linora> select utl_raw.cast_to_number('c30359') from dual;
UTL_RAW.CAST_TO_NUMBER('C30359')
--------------------------------
                           28800
--10进制转16进制
SYS@linora> select utl_raw.cast_from_number('28800') value from dual;
VALUE
------------------------------
C30359
```
#### 2.5 CHANGE # 4
```
CHANGE # 4 MEDIA RECOVERY MARKER SCN:0x0000.00000000 SEQ:0 OP:5.19 ENC:0
session number   = 261
serial  number   = 119
current username = SYS
login   username = SYS
client info      = 
OS username      = oracle
Machine name     = linora
OS terminal      = pts/1
OS process id    = 2215
OS program name  = sqlplus@linora (TNS V1-V3)
transaction name = 
version 186647552
audit sessionid 4294967295
Client Id  = 
```
第四个改变向量没有说明任何更改意图，其作用包括提供恢复操作完结点，提供有限的事后审计。  
上述update命令的目的是修改8号文件的11476块(CHANGE # 3)，但是要先在CHANGE # 1中往undo segment即3号文件(undotbs01.bdf)176号数据块注册事务表，并在事务表的的3号文件52286块保存undo data以保证读一致性及回滚，因此，在Oracle数据库中，最简单的DML指令至少要修改三个数据块。
### 3. OP Code说明
在以上的重做记录分析中，存在着一个重要的操作代码，OP=Operation Code，代表了操作的类型：  
![OP Code](/images/op1.png)
对于DML事务，Level为11，具体操作代码：  
![DML OP](/images/op2.png)
对于UNDO的操作，其代码如下所示：  
![UNDO OP](/images/op3.png)
因此，在CHANGE # 1 OP=5.2，表示更新undo segment头部信息，CHANGE # 2 OP=5.1表示undo segment头部信息，CHANGE # 3 OP=11.5表示更新操作。
### 4. 相关dump redo方法
archive log属于redo的离线，因此dump archived log语法和redo一样，参照MOS：1031381.6
```
--dump redo header
alter session set events 'immediate trace name redohdr level 10';
Session altered.
--dump an entire redo file 
alter system dump logfile '/oradata/datafile/linora/redo03.log';
--dump redo base on DBA
	--10g及以上版本：
	ALTER SYSTEM DUMP LOGFILE '/oradata/datafile/linora/redo03.log' DBA MIN 5 31125 DBA MAX 5 31150;
	--9i版本：
	ALTER SYSTEM DUMP LOGFILE '/oradata/datafile/linora/redo03.log' DBA MIN 5 . 31125 DBA MAX 5 . 31150;
--dump redo base on RBA
ALTER SYSTEM DUMP LOGFILE '/oradata/datafile/linora/redo03.log' RBA MIN 2050 . 13255 RBA MAX 2255 . 15555;
--dump redo base on SCN
alter system dump logfile '/oradata/datafile/linora/redo03.log' scn min 3109548 scn max 3109551;
--dump redo base on time
ALTER SYSTEM DUMP LOGFILE '/oradata/datafile/linora/redo03.log' TIME MIN 299425687 TIME MAX 299458800;
```
### 5. 相关函数简单说明
#### 5.1 dump
Oracle通过相应的数据库内部算法转换来进行数据存储，dump函数能查看数据在Oracle内部存储内容。
```
--用法：DUMP(expr[,number_format[,start_position][,length]]) 
--用法说明： dump(col_name,8|10|16|17) ,其中8|10|16|17 为number_format，分别为8进制，
--10进制（默认值），16进制，单字符。
SYS@linora> select dump(24000,10) from dual;
DUMP(24000,10)
---------------------
Typ=2 Len=3: 195,3,41
SYS@linora> select dump(24000,16) from dual;
DUMP(24000,16)
--------------------
Typ=2 Len=3: c3,3,29
```
#### 5.2 utl_raw
对于dump出来的内容，通常不能直观的了解到想要的信息，可以通过utl_raw来实现。以dump hr.employees员工号100所在数据块为例，来说明此函数的用法。
```
--查询数据所在DBA
SYS@linora> SELECT DBMS_ROWID.ROWID_OBJECT(rowid) "OBJECT",
DBMS_ROWID.ROWID_RELATIVE_FNO(rowid) "FILE",
  2    3  DBMS_ROWID.ROWID_BLOCK_NUMBER(rowid) "BLOCK",
  4  DBMS_ROWID.ROWID_ROW_NUMBER(rowid) "ROW"
  5  FROM hr.employees where employee_id=100;

    OBJECT       FILE      BLOCK        ROW
---------- ---------- ---------- ----------
     84737          8      11476          0
SYS@linora> alter system dump datafile 8 block 11476;
System altered.
SYS@linora> @trace
VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/linora/linora/trace/linora_ora_3141.trc
```
查看trace文件信息：
```
--表结构数据类型
SYS@linora> desc hr.employees
 Name                              Null?    Type
 --------------------------------- -------- ------------------
 EMPLOYEE_ID                       NOT NULL NUMBER(6)
 FIRST_NAME                                 VARCHAR2(20)
 LAST_NAME                         NOT NULL VARCHAR2(25)
 EMAIL                             NOT NULL VARCHAR2(25)
 PHONE_NUMBER                               VARCHAR2(20)
 HIRE_DATE                         NOT NULL DATE
 JOB_ID                            NOT NULL VARCHAR2(10)
 SALARY                                     NUMBER(8,2)
 COMMISSION_PCT                             NUMBER(2,2)
 MANAGER_ID                                 NUMBER(6)
 DEPARTMENT_ID                              NUMBER(4)
--此块中的row=0行信息，即我们查询的员工号=100的数据存储
block_row_dump:
tab 0, row 0, @0x3c3
tl: 62 fb: --H-FL-- lb: 0x2  cc: 11
col  0: [ 2]  c2 02
col  1: [ 6]  53 74 65 76 65 6e
col  2: [ 4]  4b 69 6e 67
col  3: [ 5]  53 4b 49 4e 47
col  4: [12]  35 31 35 2e 31 32 33 2e 34 35 36 37
col  5: [ 7]  78 67 06 11 01 01 01
col  6: [ 7]  41 44 5f 50 52 45 53
col  7: [ 3]  c3 03 59
col  8: *NULL*
col  9: *NULL*
col 10: [ 2]  c1 5b
```
我们找其中几个字段进行转换：
```
SYS@linora> select employee_id,LAST_NAME,SALARY from hr.employees where employee_id=100;
EMPLOYEE_ID LAST_NAME                     SALARY
----------- ------------------------- ----------
        100 King                           28800
```
#####employee_id
此字段为col 0，类型为Number，在数据库里面存储为'c2 02'，用函数转换一下：
```
SYS@linora> select utl_raw.cast_to_number('c202') employee_id from dual;
EMPLOYEE_ID
-----------
        100
```
上面的写法，当utl_raw.cast_to_number()引号中的内容多时，用起来不方便。可以加个replace()函数，处理起来便于阅读。
```
--replace函数用法：replace('exp1','exp2','exp3')从exp1中找出exp2字符串，用exp3代替
select utl_raw.cast_to_number(replace('c2 02',' ')) v from dual;
```
#####LAST_NAME
此字段存储于表中的第三个栏位，即col 2，类型为varchar2，存储数据为'4b 69 6e 67'，使用函数进行转换：
```
--转换为字符串
SYS@linora> select utl_raw.cast_to_varchar2(replace('4b 69 6e 67',' ')) last_name from dual;
LAST_NAME
----------
King
```
#####SALARY
此字段存储与表的第八个字段，即col 7，类型为Number，存储数据为'c3 03 59'，使用函数进行转换：
```
--转换为10进制数字
SYS@linora> select utl_raw.cast_to_number(replace('c3 03 59',' ')) SAL from dual;
       SAL
----------
     28800
```
同时，utl_raw还存在cast_from_number，此函数用法有点类似dump：
```
--由数字转换为16进制
SYS@linora> select utl_raw.cast_from_number(28800) SAL from dual;
SAL
--------------------
C30359

```
Reference:  
《深入浅出Oracle》--盖国强  
《Oracle Core-Essential Internals for DBAs and Developers》--Jonathan Lewis  
《临危不惧-Oracle 11g数据库恢复技术》--包光磊
