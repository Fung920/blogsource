---
layout: post
title: "Linux下编译BBED"
date: 2014-09-01 10:24:14
comments: false
categories: oracle
tags: bbed
keywords: bbed
description: 在Linux下编译BBED
---
BBED全称Block Browser and EDitor，是Oracle内部用于查看和修改数据块的工具。
<!--more-->
由于使用BBED需要对块结构有相当的了解，使用bbed风险很高，Oracle也不会对bbed提供任何技术支持，研究bbed只是为了更深入的了解Oracle的数据块结构，并不建议使用bbed去修改生产数据。本文主要示范Oracle 10g和11g版本下在Linux环境的安装。
### 1. 10g的安装
10g的安装比较简单：
```
[oracle@ora10g:/home/oracle]$ make -f $ORACLE_HOME/rdbms/lib/ins_rdbms.mk $ORACLE_HOME/rdbms/lib/bbed

Linking BBED utility (bbed)
rm -f /u01/app/oracle/product/10.2.0/db_1/rdbms/lib/bbed
gcc -o /u01/app/oracle/product/10.2.0/db_1/rdbms/lib/bbed -L/u01/app/oracle/product/10.2.0/db_1/rdbms/lib/ -L/u01/app/oracle/product/10.2.0/db_1/lib/ -L/u01/app/oracle/product/10.2.0/db_1/lib/stubs/  /u01/app/oracle/product/10.2.0/db_1/lib/s0main.o /u01/app/oracle/product/10.2.0/db_1/rdbms/lib/ssbbded.o /u01/app/oracle/product/10.2.0/db_1/rdbms/lib/sbbdpt.o `cat /u01/app/oracle/product/10.2.0/db_1/lib/ldflags`    -lnsslb10 -lncrypt10 -lnsgr10 -lnzjs10 -ln10 -lnnz10 -lnl10 /u01/app/oracle/product/10.2.0/db_1/rdbms/lib/defopt.o -ldbtools10 -lclntsh  `cat /u01/app/oracle/product/10.2.0/db_1/lib/ldflags`    -lnsslb10 -lncrypt10 -lnsgr10 -lnzjs10 -ln10 -lnnz10 -lnl10 -lnro10 `cat /u01/app/oracle/product/10.2.0/db_1/lib/ldflags`    -lnsslb10 -lncrypt10 -lnsgr10 -lnzjs10 -ln10 -lnnz10 -lnl10 -lclient10 -lnnetd10  -lvsn10 -lcommon10 -lgeneric10 -lmm -lsnls10 -lnls10  -lcore10 -lsnls10 -lnls10 -lcore10 -lsnls10 -lnls10 -lxml10 -lcore10 -lunls10 -lsnls10 -lnls10 -lcore10 -lnls10 `cat /u01/app/oracle/product/10.2.0/db_1/lib/ldflags`    -lnsslb10 -lncrypt10 -lnsgr10 -lnzjs10 -ln10 -lnnz10 -lnl10 -lnro10 `cat /u01/app/oracle/product/10.2.0/db_1/lib/ldflags`    -lnsslb10 -lncrypt10 -lnsgr10 -lnzjs10 -ln10 -lnnz10 -lnl10 -lclient10 -lnnetd10  -lvsn10 -lcommon10 -lgeneric10   -lsnls10 -lnls10  -lcore10 -lsnls10 -lnls10 -lcore10 -lsnls10 -lnls10 -lxml10 -lcore10 -lunls10 -lsnls10 -lnls10 -lcore10 -lnls10 -lclient10 -lnnetd10  -lvsn10 -lcommon10 -lgeneric10 -lsnls10 -lnls10  -lcore10 -lsnls10 -lnls10 -lcore10 -lsnls10 -lnls10 -lxml10 -lcore10 -lunls10 -lsnls10 -lnls10 -lcore10 -lnls10   `cat /u01/app/oracle/product/10.2.0/db_1/lib/sysliblist` -Wl,-rpath,/u01/app/oracle/product/10.2.0/db_1/lib -lm    `cat /u01/app/oracle/product/10.2.0/db_1/lib/sysliblist` -ldl -lm   -L/u01/app/oracle/product/10.2.0/db_1/lib
```
不妨将bbed命令加入$BIN环境变量：
```
[oracle@ora10g:/home/oracle]$ ll $ORACLE_HOME/rdbms/lib/bbed
-rwxr-xr-x 1 oracle oinstall 548768 Sep  1 10:34 /u01/app/oracle/product/10.2.0/db_1/rdbms/lib/bbed
[oracle@ora10g:/home/oracle]$ cp $ORACLE_HOME/rdbms/lib/bbed $ORACLE_HOME/bin
[oracle@ora10g:/home/oracle]$ ll $ORACLE_HOME/bin/bbed
-rwxr-xr-x 1 oracle oinstall 548768 Sep  1 10:35 /u01/app/oracle/product/10.2.0/db_1/bin/bbed
```
### 2. 11g的安装
11g及以上版本已经没有bbed相关的库文件了，需要从10g复制过来，需要复制以下几个文件：
```
[oracle@linora:/home/oracle]$ make -f $ORACLE_HOME/rdbms/lib/ins_rdbms.mk $ORACLE_HOME/rdbms/lib/bbed
...
gcc: /u01/app/oracle/product/11gr2//rdbms/lib/ssbbded.o: No such file or directory
gcc: /u01/app/oracle/product/11gr2//rdbms/lib/sbbdpt.o: No such file or directory
--从10g及以前版本copy过来以下几个文件
$ORACLE_HOME/rdbms/lib/ssbbded.o
$ORACLE_HOME/rdbms/lib/sbbdpt.o
$ORACLE_HOME/rdbms/mesg/bbedus.msb
$ORACLE_HOME/rdbms/mesg/bbedus.msg
```
编译过程跟10g的一样。
### 3. 运行bbed
bbed默认密码为blockedit，一般会使用参数文件：
```
[oracle@linora:/home/oracle]$ cat parfile.txt 
blocksize=8192
listfile=filelist.txt
mode=edit
[oracle@linora:/home/oracle]$ cat filelist.txt 
1 /oradata/datafile/linora/system01.dbf 
11 /oradata/datafile/linora/test01.dbf
```
filelist中记录的是需要查看或者修改的数据文件，第一个字段是file_id，第二个字段是file_name，第三个字段是bytes(上例中省略了)，其对应关系可用以下语句查出：
```
SYS@linora> select file#||' '||name||' '||bytes listfile from v$datafile;
LISTFILE
--------------------------------------------------------------------------------
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
以上两个文件须在同一层目录。BBED使用帮助，包括上述参数的一些说明：
```
[oracle@linora:/home/oracle]$ bbed help
LRM-00108: invalid positional parameter value 'help'
PASSWORD - Required parameter
FILENAME - Database file name
BLOCKSIZE - Database block size
LISTFILE - List file name
MODE - [browse/edit]
SPOOL - Spool to logfile [no/yes]
CMDFILE - BBED command file name
LOGFILE - BBED log file name
PARFILE - Parameter file name
BIFILE - BBED before-image file name
REVERT - Rollback changes from BIFILE [no/yes]
SILENT - Hide banner [no/yes]
HELP - Show all valid parameters [no/yes]
BBED-00105: LRM error 108 occurred during command line parsing
```
调用bbed：
```
[oracle@linora:/home/oracle]$ bbed parfile=parfile.txt 
Password: 
BBED: Release 2.0.0.0.0 - Limited Production on Mon Sep 1 11:11:15 2014
Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.
************* !!! For Oracle Internal Use only !!! ***************
BBED> 
```

Reference：  
[dissassembling_the_data_block](http://www.orafaq.com/papers/dissassembling_the_data_block.pdf)