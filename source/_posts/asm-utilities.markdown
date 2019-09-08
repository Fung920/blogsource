---
layout: post
title: "ASM utilities"
date: 2015-06-24 11:19:54
comments: false
categories: oracle
tags: asm
keywords: ASM
description: ASM Utilities,kfed,kfod,AMDU
---
Oracle自带的ASM类似OS级别的LVM，由于Oracle本身对其进行了封装，在一定程度上来讲，对维护人员就提高了要求。Oracle提供了一系列的工具能对ASM磁盘进行管理。
<!--more-->
### 1. ASMCMD
ASM由Oracle进行封装，在OS级别，我们无法访问ASM磁盘组，Oracle提供了<code>asmcmd</code>这个工具，它包含了部分类似OS级别的操作，如ls，du，cp(11g)等。在11g里面，这个工具比之前版本增强了很多。
```
--查找磁盘组信息，包括没有被mount的磁盘组
[grid@orl6:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
NAME           TARGET  STATE        SERVER                   STATE_DETAILS       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.dg
               OFFLINE OFFLINE      orl6                                         
ora.DATA1.dg
               ONLINE  ONLINE       orl6                                         
[grid@orl6:/home/grid]$ asmcmd lsdg -g  --discovery 
Inst_ID  State       Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
      1  DISMOUNTED          N           0   4096        0         0        0                0               0              0             N  DATA/
      1  MOUNTED     EXTERN  N         512   4096  1048576     20480    16970                0           16970              0             N  DATA1/
--检测ASM磁盘状态,仅被mount的磁盘能被检测
[grid@orl6:/home/grid]$ asmcmd lsdsk -k -g
Inst_ID  Total_MB  Free_MB  OS_MB  Name        Failgroup   Failgroup_Type  Library  Label  UDID  Product  Redund   Path
      1     20480    16970  20480  DATA1_0000  DATA1_0000  REGULAR         System                         UNKNOWN  /dev/asmdisk1
--检测候选磁盘状态
[grid@orl6:/home/grid]$ asmcmd lsdsk -k -g --candidate
Inst_ID  Total_MB  Free_MB  OS_MB  Name        Failgroup   Failgroup_Type  Library  Label  UDID  Product  Redund   Path
      1         0        0  12288                          REGULAR         System                         UNKNOWN  /dev/asmdisk3
```
--连接到ASM实例的客户端信息
```
[grid@orl6:/home/grid]$ asmcmd lsct
DB_Name  Status     Software_Version  Compatible_version  Instance_Name  Disk_Group
kyun     CONNECTED        11.2.0.4.0          11.2.0.4.0  kyun           DATA1     
kyun     CONNECTED        11.2.0.4.0          11.2.0.4.0  kyun           DATA      
--ASM中被打开文件句柄的文件
[grid@orl6:/home/grid]$ asmcmd lsof
DB_Name  Instance_Name  Path                                           
kyun     kyun           +data/kyun/datafile/tt01.256.883261951         
kyun     kyun           +data1/kyun/controlfile/current.256.858438947  
kyun     kyun           +data1/kyun/datafile/data.274.859116297        
kyun     kyun           +data1/kyun/datafile/demotsdata.270.858444769  
kyun     kyun           +data1/kyun/datafile/demotsidx.271.858444795   
kyun     kyun           +data1/kyun/datafile/example.264.858439059     
kyun     kyun           +data1/kyun/datafile/fung.267.858444647        
kyun     kyun           +data1/kyun/datafile/perf.268.858444681        
kyun     kyun           +data1/kyun/datafile/sysaux.261.858438997      
kyun     kyun           +data1/kyun/datafile/system.260.858438969      
kyun     kyun           +data1/kyun/datafile/test.269.858444735        
kyun     kyun           +data1/kyun/datafile/undotbs2.277.860582391    
kyun     kyun           +data1/kyun/datafile/undotbs3.278.860582395    
kyun     kyun           +data1/kyun/datafile/users.265.858439071       
kyun     kyun           +data1/kyun/onlinelog/group_1.257.858438949    
kyun     kyun           +data1/kyun/onlinelog/group_2.258.858438955    
kyun     kyun           +data1/kyun/onlinelog/group_3.259.858438959    
kyun     kyun           +data1/kyun/tempfile/temp.263.858439035               
```
11G新增cp命令
```
[grid@linora:/tmp]$ asmcmd find +data1/ *.dbf |xargs -i asmcmd cp {} /tmp
copying +data1/kyun/arch/1_56_858438943.dbf -> /tmp/1_56_858438943.dbf
copying +data1/kyun/arch/1_57_858438943.dbf -> /tmp/1_57_858438943.dbf
copying +data1/kyun/arch/1_58_858438943.dbf -> /tmp/1_58_858438943.dbf
```
其他详细的用法请参照<code>asmcmd help</code>。
### 2. KFOD
Kernal Files OSM Disk, OSM表示Order and Service Management。这个工具是在OS级别Discover Disk的。具体的用法参照kfod -h。
```
--Discover Disks
[grid@orl6:/home/grid]$ kfod status=TRUE asm_diskstring='/dev/asmdisk*' disk=all dscvgroup=TRUE
--------------------------------------------------------------------------------
 Disk          Size Header    Path                                     Disk Group   User     Group   
================================================================================
   1:      20480 Mb MEMBER    /dev/asmdisk1                            DATA1        grid     oinstall
   2:      15584 Mb MEMBER    /dev/asmdisk2                            DATA         grid     oinstall
   3:      12288 Mb CANDIDATE /dev/asmdisk3                            #            grid     oinstall
--------------------------------------------------------------------------------
ORACLE_SID ORACLE_HOME                                                          
================================================================================
      +ASM /u01/app/11gr2/grid                                                  
[grid@orl6:/home/grid]$ kfod op=groups
--------------------------------------------------------------------------------
Group          Size          Free Redundancy Name           
================================================================================
   1:      20480 Mb      16970 Mb     EXTERN DATA1          
   2:      15584 Mb      15431 Mb     EXTERN DATA                    	 
```
### 3. KFED
Kfed，全称为Kernal Files metadata EDitor。顾名思义，它是可以分析ASM磁盘头信息的，其最大的作用是能修正corrupt的ASM metadata。在11g中，这个工具随着GI的安装自动安装，但是在以前的版本中，kfed需要通过编译才能使用，如：
```
--Linux编译kfed工具before 11g
cd $ORACLE_HOME/rdbms/lib
make -f ins_rdbms.mk ikfed
```
这个工具的使用帮助可以直接在grid用户下执行kfed即可。
#### 3.1 kfed read
kfed read命令能读取单独一个ASM块。对于kfed read的语法如下：
```
kfed read [aun=ii aus=jj blkn=kk dev=]asm_disk_name
```
其中，aun表示从哪个AU(Allocation Unit)开始读取，默认为AU0，或者是ASM磁盘最开始的地方；aus表示AU大小，默认为1048576，即1M，如果ASM diskgroup使用的不是默认的AU size，请指定此大小；blkn读取第几个块，默认为0，即AU的第一个block。
```
[grid@node1:/home/grid]$ kfed read aun=0 aus=1048576 blkn=0 dev=/dev/asm-diskc
--以上命令等同于kfed read /dev/asm-diskc
--kfbh=kernal file block header
[grid@orl6:/home/grid]$ kfed read aun=0 aus=1048576 blkn=0 dev=/dev/asmdisk1
kfbh.endian:                          1 ; 0x000: 0x01				
			--系统endian，1为little，0为big
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD	
			--块类型，这里表示file header，文件头
kfbh.datfmt:                          1 ; 0x003: 0x01
kfbh.block.blk:                       0 ; 0x004: blk=0				
			--当前块位置
kfbh.block.obj:              2147483648 ; 0x008: disk=0
kfbh.check:                  3459712326 ; 0x00c: 0xce370546
kfbh.fcn.base:                     7316 ; 0x010: 0x00001c94
kfbh.fcn.wrap:                        0 ; 0x014: 0x00000000
kfbh.spare1:                          0 ; 0x018: 0x00000000
kfbh.spare2:                          0 ; 0x01c: 0x00000000
kfdhdb.driver.provstr:         ORCLDISK ; 0x000: length=8			
			--ORCLDISK+[ASM Disk Name]表示使用ASMLIB，只有ORCLDISK表示没有使用ASMLIB
kfdhdb.driver.reserved[0]:            0 ; 0x008: 0x00000000
kfdhdb.driver.reserved[1]:            0 ; 0x00c: 0x00000000
kfdhdb.driver.reserved[2]:            0 ; 0x010: 0x00000000
kfdhdb.driver.reserved[3]:            0 ; 0x014: 0x00000000
kfdhdb.driver.reserved[4]:            0 ; 0x018: 0x00000000
kfdhdb.driver.reserved[5]:            0 ; 0x01c: 0x00000000
kfdhdb.compat:                186646528 ; 0x020: 0x0b200000
			--打开本磁盘组所需的ASM最小版本，b2=11.2
kfdhdb.dsknum:                        0 ; 0x024: 0x0000
kfdhdb.grptyp:                        1 ; 0x026: KFDGTP_EXTERNAL
			--group redundancy type
kfdhdb.hdrsts:                        3 ; 0x027: KFDHDR_MEMBER
			--ASM disk header status，v$asm_disk.header_status
kfdhdb.dskname:              DATA1_0000 ; 0x028: length=10
			--ASM disk name
kfdhdb.grpname:                   DATA1 ; 0x048: length=5
			--ASM disk group name
kfdhdb.fgname:               DATA1_0000 ; 0x068: length=10
			--ASM failure group name
kfdhdb.capname:                         ; 0x088: length=0
kfdhdb.crestmp.hi:             33007118 ; 0x0a8: HOUR=0xe DAYS=0x10 MNTH=0x9 YEAR=0x7de
kfdhdb.crestmp.lo:           2136205312 ; 0x0ac: USEC=0x0 MSEC=0xfa SECS=0x35 MINS=0x1f
kfdhdb.mntstmp.hi:             33020693 ; 0x0b0: HOUR=0x15 DAYS=0x18 MNTH=0x6 YEAR=0x7df
kfdhdb.mntstmp.lo:            446956544 ; 0x0b4: USEC=0x0 MSEC=0x101 SECS=0x2a MINS=0x6
			--以上为四个时间，asm磁盘加入磁盘组的日期时间和最后被mount日期的时间
kfdhdb.secsize:                     512 ; 0x0b8: 0x0200
kfdhdb.blksize:                    4096 ; 0x0ba: 0x1000
			--ASM Metadata Block size，为4K
kfdhdb.ausize:                  1048576 ; 0x0bc: 0x00100000
			--ASM AU大小
kfdhdb.mfact:                    113792 ; 0x0c0: 0x0001bc80
kfdhdb.dsksize:                   20480 ; 0x0c4: 0x00005000
			--ASM磁盘大小，此处为20G
kfdhdb.pmcnt:                         2 ; 0x0c8: 0x00000002
kfdhdb.fstlocn:                       1 ; 0x0cc: 0x00000001
kfdhdb.altlocn:                       2 ; 0x0d0: 0x00000002
kfdhdb.f1b1locn:                      2 ; 0x0d4: 0x00000002
kfdhdb.redomirrors[0]:                0 ; 0x0d8: 0x0000
kfdhdb.redomirrors[1]:                0 ; 0x0da: 0x0000
kfdhdb.redomirrors[2]:                0 ; 0x0dc: 0x0000
kfdhdb.redomirrors[3]:                0 ; 0x0de: 0x0000
kfdhdb.dbcompat:              168820736 ; 0x0e0: 0x0a100000
			--打开磁盘组所需的数据库实例最小版本，a1=10.1
kfdhdb.grpstmp.hi:             33007118 ; 0x0e4: HOUR=0xe DAYS=0x10 MNTH=0x9 YEAR=0x7de
kfdhdb.grpstmp.lo:           2136086528 ; 0x0e8: USEC=0x0 MSEC=0x86 SECS=0x35 MINS=0x1f
			--以上两个items表示磁盘组创建时间
kfdhdb.vfstart:                       0 ; 0x0ec: 0x00000000
kfdhdb.vfend:                         0 ; 0x0f0: 0x00000000
kfdhdb.spfile:                       58 ; 0x0f4: 0x0000003a
			--ASM Spfile的AU Number，11.2以后支持，此处表示spfile存放在当前磁盘的第58AU上。
kfdhdb.spfflg:                        1 ; 0x0f8: 0x00000001
kfdhdb.ub4spare[0]:                   0 ; 0x0fc: 0x00000000
kfdhdb.ub4spare[1]:                   0 ; 0x100: 0x00000000
...
kfdhdb.acdb.aba.seq:                  0 ; 0x1d4: 0x00000000
kfdhdb.acdb.aba.blk:                  0 ; 0x1d8: 0x00000000
kfdhdb.acdb.ents:                     0 ; 0x1dc: 0x0000
kfdhdb.acdb.ub2spare:                 0 ; 0x1de: 0x0000
```
如果需要将输出保存到一个文件中，则使用text关键字，屏幕则不会输出:
```
[grid@orl6:/home/grid]$ kfed read aun=0 aus=1048576 blkn=0 dev=/dev/asmdisk1 text=/tmp/kfed.txt
[grid@orl6:/home/grid]$ more /tmp/kfed.txt 
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD
kfbh.datfmt:                          1 ; 0x003: 0x01
kfbh.block.blk:                       0 ; 0x004: blk=0
kfbh.block.obj:              2147483648 ; 0x008: disk=0
kfbh.check:                  3459712326 ; 0x00c: 0xce370546
...
```
#### 3.2 kfed备份修复ASM磁盘
kred中可以看出部分基本信息，比如kfbh.type为KFBTYP_DISKHEAD表示这个块是file header，kfdhdb.secsize,kfdhdb.blksize,kfdhdb.ausize分别表示sector size，block size和AU size。本例中AU大小为1M，块大小为4096，即4K。因此，每个AU最多有1m/4k = 256个块。
#####备份磁盘信息
手动备份磁盘头信息可采用kfed或者dd：
```
[grid@orl6:/home/grid]$ kfed read /dev/asmdisk1 aun=0 blkn=0 text=./au1.header
[grid@orl6:/home/grid]$ dd if=/dev/asmdisk1 of=./au1_bak.header  bs=4096 count=1 
```
模拟破坏磁盘头
```
[grid@orl6:/home/grid]$ dd if=/dev/zero of=/dev/asmdisk1  bs=4096 count=1
```
用kfed read查看此块(au0 block 0)：
```
[grid@orl6:/home/grid]$ kfed read aun=0 blkn=0 dev=/dev/asmdisk1 |more
kfbh.endian:                          0 ; 0x000: 0x00
kfbh.hard:                            0 ; 0x001: 0x00
kfbh.type:                            0 ; 0x002: KFBTYP_INVALID
kfbh.datfmt:                          0 ; 0x003: 0x00
kfbh.block.blk:                       0 ; 0x004: blk=0
kfbh.block.obj:                       0 ; 0x008: file=0
kfbh.check:                           0 ; 0x00c: 0x00000000
kfbh.fcn.base:                        0 ; 0x010: 0x00000000
kfbh.fcn.wrap:                        0 ; 0x014: 0x00000000
kfbh.spare1:                          0 ; 0x018: 0x00000000
kfbh.spare2:                          0 ; 0x01c: 0x00000000
7F5687C67400 00000000 00000000 00000000 00000000  [................]
  Repeat 255 times
KFED-00322: Invalid content encountered during block traversal: [kfbtTraverseBlock][Invalid OSM block type][][0]
```
已经报错了，同时，kfbh.type也变成了KFBTYP_INVALID。进行修复动作：
```
[grid@orl6:/home/grid]$ kfed repair /dev/asmdisk1
[grid@orl6:/home/grid]$ kfed read aun=0 blkn=0 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD
kfbh.datfmt:                          1 ; 0x003: 0x01
kfbh.block.blk:                       0 ; 0x004: blk=0
kfbh.block.obj:              2147483648 ; 0x008: disk=0
kfbh.check:                  3459712326 ; 0x00c: 0xce370546
kfbh.fcn.base:                     7316 ; 0x010: 0x00001c94
kfbh.fcn.wrap:                        0 ; 0x014: 0x00000000
kfbh.spare1:                          0 ; 0x018: 0x00000000
kfbh.spare2:                          0 ; 0x01c: 0x00000000
kfdhdb.driver.provstr:         ORCLDISK ; 0x000: length=8
```
通过kfed repair，kfbh.type已经变成KFBTYP_DISKHEAD,说明此block恢复正常了。   
采用kfed write进行修复：
```
[grid@orl6:/home/grid]$ dd if=/dev/zero of=/dev/asmdisk1  bs=4096 count=1
1+0 records in
1+0 records out
4096 bytes (4.1 kB) copied, 0.00122345 s, 3.3 MB/s
[grid@orl6:/home/grid]$ kfed read aun=0 blkn=0 dev=/dev/asmdisk1 |more
kfbh.endian:                          0 ; 0x000: 0x00
kfbh.hard:                            0 ; 0x001: 0x00
kfbh.type:                            0 ; 0x002: KFBTYP_INVALID
[grid@orl6:/home/grid]$ kfed write aun=0 blkn=0 dev=/dev/asmdisk1 text=./au1.header chksum=yes
[grid@orl6:/home/grid]$ kfed read aun=0 blkn=0 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD
```
采用dd进行修复：
```
[grid@orl6:/home/grid]$ dd if=./au1_bak.header  bs=4096  count=1 of=/dev/asmdisk1  bs=4096 count=1
1+0 records in
1+0 records out
4096 bytes (4.1 kB) copied, 0.00149188 s, 2.7 MB/s
[grid@orl6:/home/grid]$ kfed read aun=0 blkn=0 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD
```
需要注意的是，kfed repair仅能修复au0，后面的au，repair无效。请看下例：
```
--以AU1为例，破坏前先备份此AU数据。
[grid@orl6:/home/grid]$ dd if=/dev/asmdisk1 of=./au1_bak  bs=4096 count=1 seek=510
[grid@orl6:/home/grid]$ kfed read /dev/asmdisk1 aun=1 blkn=254 text=au1.bak
--开始破坏
[grid@orl6:/home/grid]$ dd if=/dev/zero of=/dev/asmdisk1  bs=4096 count=1 seek=510
```
确认此block状态
```
[grid@orl6:/home/grid]$ kfed read /dev/asmdisk1 aun=1 blkn=254 |more
kfbh.endian:                          0 ; 0x000: 0x00
kfbh.hard:                            0 ; 0x001: 0x00
kfbh.type:                            0 ; 0x002: KFBTYP_INVALID
```
尝试使用repair修复：
```
[grid@orl6:/home/grid]$ kfed repair /dev/asmdisk1 
KFED-00320: Invalid block num1 = [0], num2 = [1], error = [endian_kfbh]
```
尝试用dd修复：
```
[grid@orl6:/home/grid]$ dd if=./au1_bak of=/dev/asmdisk1  bs=4096 count=1  seek=510
```
查看修复结果：
```
[grid@orl6:/home/grid]$ kfed read /dev/asmdisk1 aun=1 blkn=254 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD
```
此外，还可以使用kfed write进行修复，和dd修复差不多，都要有之前的备份信息。
```
--备份
kfed read /dev/asmdisk1 aun=1 blkn=254 text=./au1.text
--破坏
dd if=/dev/zero of=/dev/asmdisk1  bs=4096 count=1 seek=510
--修复
kfed write aun=1 blkn=254 dev=/dev/asmdisk1 text=./au1.text chksum=yes
```
#### 3.3 ASM磁盘头的自动备份
对于ASM disk Header，在版本11.1.0.7以后，Oracle会自动备份一个相同的disk Header到AU1上倒数第二个块上，块的位置根据AU的大小不同而异，这个Block Number的计算可以参照以下脚本(由于ASM的默认块大小为4K，因此在1M的AU中，1024K/4K=256个block，从0开始计算，倒数第二个即为254)。在10.2.0.5版本中，ASM disk header的自动备份也已经开始存储，11gr2和10.2.0.5的备份有些许的区别，10.2.0.5中，block 0和block 254是完全一致的，包括block Number，而在11gr2中，block Number是不一致的，且Checksum值在11gr2中也是不相同的。在恢复的时候或许有点区别。
```
[grid@orl6:/home/grid]$ cat asm_dsk_hdr_bk.sh 
#!/bin/bash
echo -n "Enter the value of ASM Disk:"
read ASM_DISK
ausize=`kfed read $ASM_DISK | grep ausize | tr -s ' ' | cut -d' ' -f2`
blksize=`kfed read $ASM_DISK | grep blksize | tr -s ' ' | cut -d' ' -f2`
let n=$ausize/$blksize-2
echo "ASM Disk Header is in Block:$n"
```
##### 10.2.0.5 ASM Disk Header自动备份
```
[oracle@ora10g:/home/oracle/asm]$ ./asm_hdr_blk.sh 
Enter the value of ASM Disk:/dev/oracleasm/disks/DATA1
The NO. of ASM Header Block is:254
[oracle@ora10g:/home/oracle/asm]$ kfed read aun=1 blkn=254 dev=/dev/oracleasm/disks/DATA1 \
text=./orig_block.txt
[oracle@ora10g:/home/oracle/asm]$ kfed read aun=0 blkn=0 dev=/dev/oracleasm/disks/DATA1 \
text=./orig_block.txt
[oracle@ora10g:/home/oracle/asm]$ diff orig_block.txt bak_block.txt 
[oracle@ora10g:/home/oracle/asm]$ more bak_block.txt 
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            1 ; 0x002: KFBTYP_DISKHEAD
kfbh.datfmt:                          1 ; 0x003: 0x01
kfbh.block.blk:                       0 ; 0x004: T=0 NUMB=0x0--哪怕是备份的块，其块号和原始块号都是一样的
kfbh.block.obj:              2147483648 ; 0x008: TYPE=0x8 NUMB=0x0
kfbh.check:                   880638603 ; 0x00c: 0x347d7a8b --用于块一致性检查的校验和也相同 
```
##### 11gr2 ASM Disk Header自动备份
```
[grid@orl6:/home/grid]$ kfed read aun=0 blkn=0 dev=/dev/asmdisk1 text=./orgi_blk
[grid@orl6:/home/grid]$ kfed read aun=1 blkn=254 dev=/dev/asmdisk1 text=./bak_blk
[grid@orl6:/home/grid]$ diff orgi_blk bak_blk 
5c5
< kfbh.block.blk:                       0 ; 0x004: blk=0
---
> kfbh.block.blk:                     254 ; 0x004: blk=254
7c7
< kfbh.check:                  1002445176 ; 0x00c: 0x3bc01978
---
> kfbh.check:                  1002445190 ; 0x00c: 0x3bc01986
```
可以看到，在11gr2中，上述两个地方都是不相同的。因此，在AU1上的第254个Block将会是Au0，Block 0的备份，两个块的内容基本是相同的。在ASM实例上，可用以下命令检测DG的disk header备份信息是否一致,检查结果会出现在alert.log里面：
```
SQL> alter diskgroup data1 check 
NOTE: starting check of diskgroup DATA1
Thu Jun 25 12:22:08 2015
GMON checking disk 0 for group 1 at 9 for pid 17, osid 22337
SUCCESS: check of diskgroup DATA1 found no errors
SUCCESS: alter diskgroup data1 check
```
注意:AU1最后一个block是磁盘心跳：
```
[grid@orl6:/home/grid]$ kfed read aun=1 blkn=255 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                           19 ; 0x002: KFBTYP_HBEAT
kfbh.datfmt:                          2 ; 0x003: 0x02
kfbh.block.blk:                     511 ; 0x004: blk=511
kfbh.block.obj:              2147483648 ; 0x008: disk=0
kfbh.check:                  2462310944 ; 0x00c: 0x92c3e220
kfbh.fcn.base:                        0 ; 0x010: 0x00000000
kfbh.fcn.wrap:                        0 ; 0x014: 0x00000000
kfbh.spare1:                          0 ; 0x018: 0x00000000
kfbh.spare2:                          0 ; 0x01c: 0x00000000
kfdpHbeatB.instance:                  1 ; 0x000: 0x00000001
kfdpHbeatB.ts.hi:              33020716 ; 0x004: HOUR=0xc DAYS=0x19 MNTH=0x6 YEAR=0x7df
kfdpHbeatB.ts.lo:            1435727872 ; 0x008: USEC=0x0 MSEC=0xde SECS=0x19 MINS=0x15
kfdpHbeatB.rnd[0]:            148639519 ; 0x00c: 0x08dc0f1f
kfdpHbeatB.rnd[1]:           4128301322 ; 0x010: 0xf610e10a
kfdpHbeatB.rnd[2]:           1569681180 ; 0x014: 0x5d8f6f1c
kfdpHbeatB.rnd[3]:           3891741690 ; 0x018: 0xe7f743fa
```
### 4. AMDU--ASM Metadata Dump Utility 
AMDU在磁盘组无法挂载的时候，是一个很好的抽取数据攻击。磁盘组是否mounted跟amdu并没有关系，amdu可以处理dismounted状态的磁盘组。其主要功能是从ASM disk中抽取metadata。这个工具是11g以后的，10g如果要用到可以到MOS上去下载。AMDU的用法帮助可以使用<code>amdu help=y</code>查看。   
amdu可以查看磁盘组详细的信息，如：
```
[grid@orl6:/home/grid]$ amdu -diskstring '/dev/asm*' -dump 'DATA1'
```
下面模拟ASM磁盘组无法挂载的情况下抽取ASM里面相关的文件，如数据文件、控制文件，甚至参数文件。   
在上述的条件下，我们首先要确认各种文件的file number，这些文件的file number及alias都是存储ASM中，因此，首先要确认ASM Alias在哪个AU，而指向alias 所在的AU为file directory。而在ASM AU分布中，AU0和AU1为physical metadata，ASM的File directory在AU2，block 6上，首先通过file directory找出alias所在AU。
```
[grid@orl6:/home/grid]$ kfed read aun=2 blkn=6 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                            4 ; 0x002: KFBTYP_FILEDIR
...
kfffde[0].xptr.au:                   48 ; 0x4a0: 0x00000030
..
```
xptr.au即alias directory所在的au,从blkn=0开始查找，本例中：
```
#spfile的file number
[grid@orl6:/home/grid]$ kfed read aun=48 blkn=3 dev=/dev/asmdisk1 |more
kfade[5].name:           spfilekyun.ora ; 0x1b0: length=14		
kfade[5].fnum:                      266 ; 0x1e0: 0x0000010a		
#controlfile的file number
[grid@orl6:/home/grid]$ kfed read aun=48 blkn=4 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                           11 ; 0x002: KFBTYP_ALIASDIR
...
...kfade[0].name:                  Current ; 0x034: length=7
kfade[0].fnum:                      256 ; 0x064: 0x00000100		
...
#online redo log的file number
[grid@orl6:/home/grid]$ kfed read aun=48 blkn=5 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                           11 ; 0x002: KFBTYP_ALIASDIR
...
kfade[0].name:                  group_1 ; 0x034: length=7
kfade[0].fnum:                      257 ; 0x064: 0x00000101
...
kfade[1].name:                  group_2 ; 0x080: length=7
kfade[1].fnum:                      258 ; 0x0b0: 0x00000102
...
kfade[2].name:                  group_3 ; 0x0cc: length=7
kfade[2].fnum:                      259 ; 0x0fc: 0x00000103		--online redo log的file number
...
#Tablespace的file number
[grid@orl6:/home/grid]$ kfed read aun=48 blkn=6 dev=/dev/asmdisk1 |more
kfbh.endian:                          1 ; 0x000: 0x01
kfbh.hard:                          130 ; 0x001: 0x82
kfbh.type:                           11 ; 0x002: KFBTYP_ALIASDIR
...
kfade[0].name:                   SYSTEM ; 0x034: length=6
kfade[0].fnum:                      260 ; 0x064: 0x00000104
...
kfade[1].name:                   SYSAUX ; 0x080: length=6
kfade[1].fnum:                      261 ; 0x0b0: 0x00000105
...
```
通过kfed得到file number，然后使用amdu进行抽取即可。在实际环境中，spfile和control file的file number其实是可以通过alert日志得到相关的信息的，有了control file，即可以读取datafile，从<code>v$datafile</code>中可以获取datafile的file number。
```
#使用amdu抽取spfile
[grid@orl6:/home/grid]$ amdu -diskstring='/dev/asmdisk1' -extract DATA1.266 -nodir -noreport -output kyun.spfile.266
[grid@orl6:/home/grid]$ strings kyun.spfile.266 
kyun.__db_cache_size=364904448
kyun.__java_pool_size=4194304
kyun.__large_pool_size=8388608
kyun.__oracle_base='/u01/app/oracle'#ORACLE_BASE set from environment
kyun.__pga_aggregate_target=322961408
kyun.__sga_target=515899392
kyun.__shared_io_pool_size=0
kyun.__shared_pool_size=130023424
kyun.__streams_pool_size=0
*.audit_file_dest='/u01/app/oracle/admin/kyun/adump'
*.audit_trail='db'
*.compatible='11.2.0.4.0'
*.control_files='+DATA1/kyun/controlfile/current.256.858438947'
*.db_block_size=8192
*.db_create_file_dest='+DATA1'
*.db_domain=''
*.db_name='kyun'
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=kyunXDB)'
*.log_archive_dest_1='location=+DATA1/kyun/arch'
*.memory_target=838860800
*.open_cursors=300
*.processes=150
*.remote_login_passwordfile='EXCLUSIVE'
*.undo_tablespace='UNDOTBS1'
```
修改control_files路径，启动数据库至mount状态，查询数据文件，同样抽取出数据文件，通过rename或者重建控制文件即可把库拉起来。

```
[grid@orl6:/home/grid]$ amdu -dis='/dev/asm*' -extract DATA1.267 -nodir -noreport -output fung.267.dbf
[grid@orl6:/home/grid]$ dbv
dbv   dbvO  
[grid@orl6:/home/grid]$ dbv file=fung.267.dbf 

DBVERIFY: Release 11.2.0.4.0 - Production on Thu Jun 25 13:14:43 2015

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /home/grid/fung.267.dbf


DBVERIFY - Verification complete

Total Pages Examined         : 37376
Total Pages Processed (Data) : 24705
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 5545
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 771
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 6355
Total Pages Marked Corrupt   : 0
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 748761 (0.748761)
```

Reference:   
[ID 1485597.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1485597.1)  

</br>
<b>EOF</b>
</br>

