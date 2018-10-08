---
layout: post
title: "BBED Structure"
date: 2014-09-01 16:07:05
comments: false
categories: oracle
tags: bbed
keywords: bbed
description: bbed中的块结构
---
BBED中map命令能显示数据块的数据结构，本文以bbed分析数据块头及数据块为例，说明map中的具体结构。
<!--more-->
### 1. 数据块头结构
```
BBED> set dba 1,1
        DBA             0x00400001 (4194305 1,1)

BBED> map /v
 File: /oradata/datafile/linora/system01.dbf (1)
 Block: 1                                     Dba:0x00400001
------------------------------------------------------------
 Data File Header									--表示文件头

 struct kcvfh, 860 bytes                    @0       --此块只有一个Structure，kcvfh
    struct kcvfhbfh, 20 bytes               @0       
    struct kcvfhhdr, 76 bytes               @20      
    ub4 kcvfhrdb                            @96      
    struct kcvfhcrs, 8 bytes                @100     
    ub4 kcvfhcrt                            @108     
    ub4 kcvfhrlc                            @112     
    struct kcvfhrls, 8 bytes                @116     
    ub4 kcvfhbti                            @124     
    struct kcvfhbsc, 8 bytes                @128     
    ub2 kcvfhbth                            @136     
    ub2 kcvfhsta                            @138     
    struct kcvfhckp, 36 bytes               @484     
    ub4 kcvfhcpc                            @140     
    ub4 kcvfhrts                            @144     
    ub4 kcvfhccc                            @148     
    struct kcvfhbcp, 36 bytes               @152     
    ub4 kcvfhbhz                            @312     
    struct kcvfhxcd, 16 bytes               @316     
    sword kcvfhtsn                          @332     
    ub2 kcvfhtln                            @336     
    text kcvfhtnm[30]                       @338     
    ub4 kcvfhrfn                            @368     
    struct kcvfhrfs, 8 bytes                @372     
    ub4 kcvfhrft                            @380     
    struct kcvfhafs, 8 bytes                @384     
    ub4 kcvfhbbc                            @392     
    ub4 kcvfhncb                            @396     
    ub4 kcvfhmcb                            @400     
    ub4 kcvfhlcb                            @404     
    ub4 kcvfhbcs                            @408     
    ub2 kcvfhofb                            @412     
    ub2 kcvfhnfb                            @414     
    ub4 kcvfhprc                            @416     
    struct kcvfhprs, 8 bytes                @420     
    struct kcvfhprfs, 8 bytes               @428     
    ub4 kcvfhtrt                            @444     

 ub4 tailchk                                @8188    
```
print kcvfh看看具体内容：
```
BBED> p kcvfh
struct kcvfh, 860 bytes                     @0       
   struct kcvfhbfh, 20 bytes                @0       					
   --数据块头
   --前20个字节对应如图一所示
      ub1 type_kcbh                         @0        0x0b				
	  --Header Block Type，如图二
	  --此处转换为10进制是11，正是file header
      ub1 frmt_kcbh                         @1        0xa2				
	  --块格式，1为Oracle 7,2为Oracle 8后
      ub1 spare1_kcbh                       @2        0x00				
      ub1 spare2_kcbh                       @3        0x00				
      ub4 rdba_kcbh                         @4        0x00400001		
	  --RDBA,相对块地址
      ub4 bas_kcbh                          @8        0x00000000		
	  --SCN base
      ub2 wrp_kcbh                          @12       0x0000			
	  --SCN Wrap
      ub1 seq_kcbh                          @14       0x01				
	  --序列号，同一个SCN不同的序列号，标记块版本
      ub1 flg_kcbh                          @15       0x04 (KCBHFCKV)	
	  --标记号
		--0x01新块
		--0x02 Delayed Logging Chang advanced SCN/seq
		--0x04 Check value saved
      ub2 chkval_kcbh                       @16       0x1375			
	  --块校验值
      ub2 spare3_kcbh                       @18       0x0000			
   struct kcvfhhdr, 76 bytes                @20      					
   --文件头通用数据类型
      ub4 kccfhswv                          @20       0x00000000		
      ub4 kccfhcvn                          @24       0x0b200400		
      ub4 kccfhdbi                          @28       0xc9cffd9d		
	  --dbid
      text kccfhdbn[0]                      @32      L					
	  --接下来8个字符为Oracle SID，只能<=8个字符
      text kccfhdbn[1]                      @33      I
      text kccfhdbn[2]                      @34      N
      text kccfhdbn[3]                      @35      O
      text kccfhdbn[4]                      @36      R
      text kccfhdbn[5]                      @37      A
      text kccfhdbn[6]                      @38       
      text kccfhdbn[7]                      @39       
      ub4 kccfhcsq                          @40       0x00002353		
	  --Controlfile sequence number
      ub4 kccfhfsz                          @44       0x00016800		
	  --所在数据文件块数=dba_data_files.blocks
      s_blkz kccfhbsz                       @48       0x00				
      ub2 kccfhfno                          @52       0x0001
	  --文件号file_id
      ub2 kccfhtyp                          @54       0x0003
	  --文件类型，03表示数据文件，06表示undo
      ub4 kccfhacid                         @56       0x00000000
      ub4 kccfhcks                          @60       0x00000000
      text kccfhtag[0]                      @64       
      text kccfhtag[1]                      @65       
      text kccfhtag[2]                      @66       
      text kccfhtag[3]                      @67       
      text kccfhtag[4]                      @68       
      text kccfhtag[5]                      @69       
      text kccfhtag[6]                      @70       
      text kccfhtag[7]                      @71       
      text kccfhtag[8]                      @72       
      text kccfhtag[9]                      @73       
      text kccfhtag[10]                     @74       
      text kccfhtag[11]                     @75       
      text kccfhtag[12]                     @76       
      text kccfhtag[13]                     @77       
      text kccfhtag[14]                     @78       
      text kccfhtag[15]                     @79       
      text kccfhtag[16]                     @80       
      text kccfhtag[17]                     @81       
      text kccfhtag[18]                     @82       
      text kccfhtag[19]                     @83       
      text kccfhtag[20]                     @84       
      text kccfhtag[21]                     @85       
      text kccfhtag[22]                     @86       
      text kccfhtag[23]                     @87       
      text kccfhtag[24]                     @88       
      text kccfhtag[25]                     @89       
      text kccfhtag[26]                     @90       
      text kccfhtag[27]                     @91       
      text kccfhtag[28]                     @92       
      text kccfhtag[29]                     @93       
      text kccfhtag[30]                     @94       
      text kccfhtag[31]                     @95       
   ub4 kcvfhrdb                             @96       0x00400208
   --仅1号文件有值，代表root dba，本例中转换为10进制为4194824，
   --通过dbms_utility可以查看到正是1号文件520 Block。
   --在11g中，dba 1,520 代表的是bootstrap$，10g是dba 1,377，10g以前是dba 1,417
   --查询语句：
   --col SEGMENT_NAME for a15
   --select segment_name,segment_type,header_file,header_block from dba_segments where header_block=520;
   struct kcvfhcrs, 8 bytes                 @100     
   --数据文件创建scn
      ub4 kscnbas                           @100      0x0000000e
      ub2 kscnwrp                           @104      0x0000
   ub4 kcvfhcrt                             @108      0x3201eb02
   --数据文件创建时间
   ub4 kcvfhrlc                             @112      0x32a6fb05
   --resetlog时间
   struct kcvfhrls, 8 bytes                 @116     
   --resetlog scn
      ub4 kscnbas                           @116      0x00170c23
      ub2 kscnwrp                           @120      0x0000
   ub4 kcvfhbti                             @124      0x00000000
   --begin hot backup time
   struct kcvfhbsc, 8 bytes                 @128     
   --last backup stared scn
      ub4 kscnbas                           @128      0x00000000
      ub2 kscnwrp                           @132      0x0000
   ub2 kcvfhbth                             @136      0x0000
   --begin hot backup redo thread
   ub2 kcvfhsta                             @138      0x2004 (KCVFHOFZ)
   --数据文件状态，04为正常，00为关闭，01为begin backup
   struct kcvfhckp, 36 bytes                @484     
   --检查点信息
      struct kcvcpscn, 8 bytes              @484     
	  --检查点scn
         ub4 kscnbas                        @484      0x003443f4
         ub2 kscnwrp                        @488      0x0000
      ub4 kcvcptim                          @492      0x331aa11c
	  --检查点时间
      ub2 kcvcpthr                          @496      0x0001
	  --检查点线程号
      union u, 12 bytes                     @500     
         struct kcvcprba, 12 bytes          @500     
		 --Checkpoint scn redo rda(参照前文[重做日志]关于rba的描述)
            ub4 kcrbaseq                    @500      0x000000ae
			--log序列号
            ub4 kcrbabno                    @504      0x00000002
			--块号
            ub2 kcrbabof                    @508      0x0010
			--偏移量
      ub1 kcvcpetb[0]                       @512      0x02
	  --最大线程数
      ub1 kcvcpetb[1]                       @513      0x00
      ub1 kcvcpetb[2]                       @514      0x00
      ub1 kcvcpetb[3]                       @515      0x00
      ub1 kcvcpetb[4]                       @516      0x00
      ub1 kcvcpetb[5]                       @517      0x00
      ub1 kcvcpetb[6]                       @518      0x00
      ub1 kcvcpetb[7]                       @519      0x00
   ub4 kcvfhcpc                             @140      0x00000253
   --v$datafile_header.CHECKPOINT_COUNT，数据文件发生Checkpoint的次数
   ub4 kcvfhrts                             @144      0x331aa119
   --recoved time
   ub4 kcvfhccc                             @148      0x00000252
   --控制文件检查点次数
   struct kcvfhbcp, 36 bytes                @152     
   --backup Checkpoint
      struct kcvcpscn, 8 bytes              @152     
	  --begin backup checkpoint scn 
         ub4 kscnbas                        @152      0x00000000
         ub2 kscnwrp                        @156      0x0000
      ub4 kcvcptim                          @160      0x00000000
	  --begin backup checkpoint time
      ub2 kcvcpthr                          @164      0x0000
	  --begin backup checkpoint thread
      union u, 12 bytes                     @168     
         struct kcvcprba, 12 bytes          @168     
		 --begin backup checkpoint rba
            ub4 kcrbaseq                    @168      0x00000000
            ub4 kcrbabno                    @172      0x00000000
            ub2 kcrbabof                    @176      0x0000
      ub1 kcvcpetb[0]                       @180      0x00
      ub1 kcvcpetb[1]                       @181      0x00
      ub1 kcvcpetb[2]                       @182      0x00
      ub1 kcvcpetb[3]                       @183      0x00
      ub1 kcvcpetb[4]                       @184      0x00
      ub1 kcvcpetb[5]                       @185      0x00
      ub1 kcvcpetb[6]                       @186      0x00
      ub1 kcvcpetb[7]                       @187      0x00
   ub4 kcvfhbhz                             @312      0x00000000
   struct kcvfhxcd, 16 bytes                @316     
      ub4 space_kcvmxcd[0]                  @316      0x00000000
      ub4 space_kcvmxcd[1]                  @320      0x00000000
      ub4 space_kcvmxcd[2]                  @324      0x00000000
      ub4 space_kcvmxcd[3]                  @328      0x00000000
   sword kcvfhtsn                           @332      0
   --表空间号:v$tablespace.ts$
   ub2 kcvfhtln                             @336      0x0006
   --表空间名字，最大为30字节
   text kcvfhtnm[0]                         @338     S
   text kcvfhtnm[1]                         @339     Y
   text kcvfhtnm[2]                         @340     S
   text kcvfhtnm[3]                         @341     T
   text kcvfhtnm[4]                         @342     E
   text kcvfhtnm[5]                         @343     M
   text kcvfhtnm[6]                         @344      
   text kcvfhtnm[7]                         @345      
   text kcvfhtnm[8]                         @346      
   text kcvfhtnm[9]                         @347      
   text kcvfhtnm[10]                        @348      
   text kcvfhtnm[11]                        @349      
   text kcvfhtnm[12]                        @350      
   text kcvfhtnm[13]                        @351      
   text kcvfhtnm[14]                        @352      
   text kcvfhtnm[15]                        @353      
   text kcvfhtnm[16]                        @354      
   text kcvfhtnm[17]                        @355      
   text kcvfhtnm[18]                        @356      
   text kcvfhtnm[19]                        @357      
   text kcvfhtnm[20]                        @358      
   text kcvfhtnm[21]                        @359      
   text kcvfhtnm[22]                        @360      
   text kcvfhtnm[23]                        @361      
   text kcvfhtnm[24]                        @362      
   text kcvfhtnm[25]                        @363      
   text kcvfhtnm[26]                        @364      
   text kcvfhtnm[27]                        @365      
   text kcvfhtnm[28]                        @366      
   text kcvfhtnm[29]                        @367      
   ub4 kcvfhrfn                             @368      0x00000001
   --相对文件号
   struct kcvfhrfs, 8 bytes                 @372     
   --Recovery fuzzy scn 
      ub4 kscnbas                           @372      0x00000000
      ub2 kscnwrp                           @376      0x0000
   ub4 kcvfhrft                             @380      0x00000000
   --Recovery fuzzy time 
   struct kcvfhafs, 8 bytes                 @384     
   --Absolute fuzzy scn  
      ub4 kscnbas                           @384      0x00000000
      ub2 kscnwrp                           @388      0x0000
   ub4 kcvfhbbc                             @392      0x00000000
   --Backup Block Count
   ub4 kcvfhncb                             @396      0x00000000
   --marked media Corrupt Blocks
   ub4 kcvfhmcb                             @400      0x00000000
   --Media Corrupt Blocks 
   ub4 kcvfhlcb                             @404      0x00000000
   --Logically Corrupt Blocks
   ub4 kcvfhbcs                             @408      0x00000000
   --Backup Completion timeStamp
   ub2 kcvfhofb                             @412      0x000a
   ub2 kcvfhnfb                             @414      0x000a
   ub4 kcvfhprc                             @416      0x323c286e
   --prev reset logs time 
   struct kcvfhprs, 8 bytes                 @420     
   --prev reset logs scn 
      ub4 kscnbas                           @420      0x001084e6
      ub2 kscnwrp                           @424      0x0000
   struct kcvfhprfs, 8 bytes                @428     
      ub4 kscnbas                           @428      0x00000000
      ub2 kscnwrp                           @432      0x0000
   ub4 kcvfhtrt                             @444      0x00000000
   --Terminal Recovery Stamp
```
图1 Block Structure  
![Data Block](/images/bbed_kcvfhbfh.png)
图2 Header block type  
![Header block type](/images/bbed_header_block_type.png)
### 2. 数据块结构
数据块的结构如图三所示，包括cache层，事务层和数据层。  
图3 Data Block Structure  
![data block](/images/bbed_data_block.png)
```
BBED> set dba 8,11477
        DBA             0x02002cd5 (33565909 8,11477)
BBED> map /v
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477                                 Dba:0x02002cd5
------------------------------------------------------------
 KTB Data Block (Table/Cluster)					--表示属于KTB数据块
 struct kcbh, 20 bytes                      @0     
--数据块头，其定义跟kcvfh一样 
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
 --事务层
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[2], 48 bytes            @44      
 struct kdbh, 14 bytes                      @100    
--数据层 
    ub1 kdbhflag                            @100     
    sb1 kdbhntab                            @101     
    sb2 kdbhnrow                            @102     
    sb2 kdbhfrre                            @104     
    sb2 kdbhfsbo                            @106     
    sb2 kdbhfseo                            @108     
    sb2 kdbhavsp                            @110     
    sb2 kdbhtosp                            @112     
 struct kdbt[1], 4 bytes                    @114     
 --表目录层
    sb2 kdbtoffs                            @114     
    sb2 kdbtnrow                            @116     
 sb2 kdbr[9]                                @118     
 --行目录层
 ub1 freespace[7430]                        @136     
 --空闲空间
 ub1 rowdata[622]                           @7566    
 --实际行数据
 ub4 tailchk                                @8188    
```
分别打印上述结构，看看具体内容(kcbh(block header structure)定义如kcvfh.kcvfhbfh一致，均为数据块头，此处不在赘述)。
#### 2.1 ktbbh(Transaction Fixed Header Structure)
```
BBED> p ktbbh
struct ktbbh, 72 bytes                      @20      
   ub1 ktbbhtyp                             @20       0x01 (KDDBTDATA)
   --块类型，1为data，2为index
   union ktbbhsid, 4 bytes                  @24      
   --Segment/Object ID，如果两者一致，表示对象没有被Truncate过
      ub4 ktbbhsg1                          @24       0x00014b01
      ub4 ktbbhod1                          @24       0x00014b01
   struct ktbbhcsc, 8 bytes                 @28      
   --块最后清除的SCN
      ub4 kscnbas                           @28       0x0021d786
      ub2 kscnwrp                           @32       0x0000
   sb2 ktbbhict                             @36       2
   --ITL slot number，事务槽号
   ub1 ktbbhflg                             @38       0x32 (NONE)
   --0=on the free list
   ub1 ktbbhfsl                             @39       0x00
   --ITL free list slot
   ub4 ktbbhfnx                             @40       0x02002cd0
   --DBA of next block on the freelist
   struct ktbbhitl[0], 24 bytes             @44      
   --ITL list index
      struct ktbitxid, 8 bytes              @44      
	  --ITL xid
         ub2 kxidusn                        @44       0x0003
		 --usn
         ub2 kxidslt                        @46       0x000b
		 --slot
         ub4 kxidsqn                        @48       0x0000050d
		 --Sequence
      struct ktbituba, 8 bytes              @52      
	  --uba
         ub4 kubadba                        @52       0x00c006d4
         ub2 kubaseq                        @56       0x00d2
         ub1 kubarec                        @58       0x34
      ub2 ktbitflg                          @60       0x2009 (KTBFUPB)
      union _ktbitun, 2 bytes               @62      
         sb2 _ktbitfsc                      @62       0
         ub2 _ktbitwrp                      @62       0x0000
      ub4 ktbitbas                          @64       0x0021d795
   struct ktbbhitl[1], 24 bytes             @68      
      struct ktbitxid, 8 bytes              @68      
         ub2 kxidusn                        @68       0x0000
         ub2 kxidslt                        @70       0x0000
         ub4 kxidsqn                        @72       0x00000000
      struct ktbituba, 8 bytes              @76      
         ub4 kubadba                        @76       0x00000000
         ub2 kubaseq                        @80       0x0000
         ub1 kubarec                        @82       0x00
      ub2 ktbitflg                          @84       0x0000 (NONE)
      union _ktbitun, 2 bytes               @86      
         sb2 _ktbitfsc                      @86       0
         ub2 _ktbitwrp                      @86       0x0000
      ub4 ktbitbas                          @88       0x00000000
```
#### 2.2 kdbh(Data Header Structure)
```
BBED> p kdbh
struct kdbh, 14 bytes                       @100     
   ub1 kdbhflag                             @100      0x00 (NONE)
   sb1 kdbhntab                             @101      1
   --表的个数，clusters>1
   sb2 kdbhnrow                             @102      9
   --块中包含的行数
   sb2 kdbhfrre                             @104     -1
   --是否在空闲列表，-1表示不在空闲列表
   sb2 kdbhfsbo                             @106      36
   --freespace start offset
   sb2 kdbhfseo                             @108      7466
   --freespace end offset
   sb2 kdbhavsp                             @110      7430
   --块内可用空间
   sb2 kdbhtosp                             @112      7430
   --事务提交后所有的可用空间
```
#### 2.3 kdbt(Table Directory Entry Structure)
```
BBED> p kdbt
struct kdbt[0], 4 bytes                     @114     
   sb2 kdbtoffs                             @114      0
   sb2 kdbtnrow                             @116      9
```
#### 2.4 kdbr(Row Directory)
kdbr代表行目录，后面的n是代表该块中存储了多少行数据记录，从第0行开始计算。下例中，表示该块中存储了9行数据。
```
BBED> p kdbr
sb2 kdbr[0]                                 @118      8015
sb2 kdbr[1]                                 @120      7946
sb2 kdbr[2]                                 @122      7876
sb2 kdbr[3]                                 @124      7803
sb2 kdbr[4]                                 @126      7744
sb2 kdbr[5]                                 @128      7677
sb2 kdbr[6]                                 @130      7612
sb2 kdbr[7]                                 @132      7538
sb2 kdbr[8]                                 @134      7466
```
结合*号可以打印此块中的第n行数据
```
BBED> p *kdbr[2]
rowdata[410]
------------
ub1 rowdata[410]                            @7976     0x2c
--2c代表正常为删除数据，3c代表标记为删除的数据行，第二列是该行数据的偏移量，第三列为标识符fb。
BBED> d /v
 File: /oradata/datafile/linora/perf01.dbf (8)
 Block: 11477   Offsets: 7976 to 8191  Dba:0x02002cd5
-------------------------------------------------------
 2c010b02 c203084a 414e6e69 66657206 l ,......JANnifer.
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
BBED> x /rncccctcnnnn
rowdata[410]                                @7976    
------------
flag@7976: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@7977: 0x01
cols@7978:   11

col    0[2] @7979: 200 
col    1[8] @7982: JANnifer
col    2[6] @7991: Whalen
col    3[7] @7998: JWHALEN
col   4[12] @8006: 515.123.4444
col    5[7] @8019: 2003-09-17 00:00:00 
col    6[7] @8027: AD_ASST
col    7[2] @8035: 4400 
col    8[0] @8038: *NULL*
col    9[3] @8039: 101 
col   10[2] @8043: 10 
```

Reference：  
[dissassembling_the_data_block](http://www.orafaq.com/papers/dissassembling_the_data_block.pdf)