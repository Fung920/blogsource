---
layout: post
title: "Opatch"
date: 2015-04-18 11:24:29
comments: false
categories: oracle
tags: 
keywords: optach
description: 使用opatch打小补丁
---
Opatch主要是针对小补丁集管理的工具，不同于release，这些小补丁包含PSU(每季度发布)，one-off patch，CPU等。
<!--more-->
打补丁前先将opatch升级至最新版，同时先停止数据库及监听服务。
### 1.新版opatch下载地址
[https://updates.oracle.com/download/6880880.html](https://updates.oracle.com/download/6880880.html)
### 2.查看当前数据库opatch版本
```
oracle@linux:~> $ORACLE_HOME/OPatch/opatch version
Invoking OPatch 11.2.0.1.7
OPatch Version: 11.2.0.1.7
OPatch succeeded.
```
### 3.更新至最新版opatch
```
oracle@linux:/opt> cd ~/opatch/
oracle@linux:~/opatch> ll
total 32668
-rw-r--r-- 1 root root 33020933 Feb 10 15:47 p6880880_112000_Linux-x86-64.zip
#修改属性
linux:~ # chown -R oracle:oinstall /home/oracle/opatch/
#解压zip包
oracle@linux:~/opatch> ll
total 32668
-rw-r--r-- 1 oracle oinstall 33020933 Feb 10 15:47 p6880880_112000_Linux-x86-64.zip
oracle@linux:~/opatch> unzip p6880880_112000_Linux-x86-64.zip 
oracle@linux:~/opatch> ls -ltr
total 32672
drwxr-xr-x 8 oracle oinstall     4096 Dec 14  2013 OPatch
-rw-r--r-- 1 oracle oinstall 33020933 Feb 10 15:47 p6880880_112000_Linux-x86-64.zip
```
解压完直接替换即可：
```
oracle@linux:~/opatch> cd /oracle/app/oracle/product/11.2.0/linoradb
oracle@linux:/oracle/app/oracle/product/11.2.0/linoradb> mv OPatch OPatch_old
oracle@linux:/oracle/app/oracle/product/11.2.0/linoradb> mv /home/oracle/opatch/OPatch/ \
/oracle/app/oracle/product/11.2.0/linoradb/
#查看更新后版本
oracle@linux:/oracle/app/oracle/product/11.2.0/linoradb> OPatch/opatch version
OPatch Version: 11.2.0.3.6
OPatch succeeded.
```
### 4.安装前准备
--当前数据库PSU及补丁情况
```
[oracle@linora:/worktmp]$ $ORACLE_HOME/OPatch/opatch lsinv
Oracle Interim Patch Installer version 11.2.0.3.6
Copyright (c) 2013, Oracle Corporation.  All rights reserved.

Oracle Home       : /u01/app/oracle/product/11gr2
Central Inventory : /u01/app/oraInventory
   from           : /u01/app/oracle/product/11gr2/oraInst.loc
OPatch version    : 11.2.0.3.6
OUI version       : 11.2.0.3.0
Log file location : /u01/app/oracle/product/11gr2/cfgtoollogs/opatch/opatch2015-03-25_09-34-31AM_1.log

Lsinventory Output file location : 
	/u01/app/oracle/product/11gr2/cfgtoollogs/opatch/lsinv/lsinventory2015-03-25_09-34-31AM.txt

--------------------------------------------------------------------------------
Installed Top-level Products (1): 

Oracle Database 11g                                                  11.2.0.3.0
There are 1 product(s) installed in this Oracle Home.

Interim patches (1) :

Patch  13696216     : applied on Wed Mar 25 09:25:43 CST 2015
Unique Patch ID:  14596729
Patch description:  "Database Patch Set Update : 11.2.0.3.2 (13696216)"
   Created on 3 Apr 2012, 22:02:51 hrs PST8PDT
Sub-patch  13343438; "Database Patch Set Update : 11.2.0.3.1 (13343438)"
   Bugs fixed:
     13070939, 13035804, 10350832, 13632717, 13041324, 12919564, 13420224
     13742437, 12861463, 12834027, 13742438, 13332439, 13036331, 13499128
     12998795, 12829021, 13492735, 9873405, 13742436, 13503598, 12960925
     12718090, 13742433, 12662040, 9703627, 12905058, 12938841, 13742434
     12849688, 12950644, 13362079, 13742435, 12620823, 12917230, 12845115
     12656535, 12764337, 13354082, 12588744, 11877623, 12612118, 12847466
     13742464, 13528551, 12894807, 13343438, 12582664, 12780983, 12748240
     12797765, 12780098, 13696216, 12923168, 13466801, 13772618, 11063191, 13554409
```
以上列出了已经安装了的补丁集：13696216,13343438。
```
--解压11.2.0.3.4
[oracle@linora:/worktmp]$ unzip p14275605_112030_Linux-x86-64.zip
[oracle@linora:/worktmp]$ cd 14275605/
--检查11.2.0.3.4是否与之前11.2.0.3.2有冲突
[oracle@linora:/worktmp/14275605]$ $ORACLE_HOME/OPatch/opatch prereq \
CheckConflictAgainstOHWithDetail -ph ./
Oracle Interim Patch Installer version 11.2.0.3.6
Copyright (c) 2013, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/app/oracle/product/11gr2
Central Inventory : /u01/app/oraInventory
   from           : /u01/app/oracle/product/11gr2/oraInst.loc
OPatch version    : 11.2.0.3.6
OUI version       : 11.2.0.3.0
Log file location : /u01/app/oracle/product/11gr2/cfgtoollogs/opatch/opatch2015-03-25_09-37-00AM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
```
### 5.安装psu
关闭数据库相关服务，如EM，OGG等，在此之前，要确保对数据库做了备份。
进入patch 目录，直接进行apply：
```
[oracle@linora:/worktmp/14275605]$ $ORACLE_HOME/OPatch/opatch apply
```
### 6.确认补丁状况
```
[oracle@linora:/worktmp/14275605]$ $ORACLE_HOME/OPatch/opatch lsinv
Oracle Interim Patch Installer version 11.2.0.3.6
Copyright (c) 2013, Oracle Corporation.  All rights reserved.

Oracle Home       : /u01/app/oracle/product/11gr2
Central Inventory : /u01/app/oraInventory
   from           : /u01/app/oracle/product/11gr2/oraInst.loc
OPatch version    : 11.2.0.3.6
OUI version       : 11.2.0.3.0
Log file location : /u01/app/oracle/product/11gr2/cfgtoollogs/opatch/opatch2015-03-25_10-12-01AM_1.log

Lsinventory Output file location : 
/u01/app/oracle/product/11gr2/cfgtoollogs/opatch/lsinv/lsinventory2015-03-25_10-12-01AM.txt

--------------------------------------------------------------------------------
Installed Top-level Products (1): 

Oracle Database 11g                                                  11.2.0.3.0
There are 1 product(s) installed in this Oracle Home.

Interim patches (1) :

Patch  14275605     : applied on Wed Mar 25 09:55:51 CST 2015
Unique Patch ID:  15367368
Patch description:  "Database Patch Set Update : 11.2.0.3.4 (14275605)"
   Created on 3 Oct 2012, 18:38:19 hrs PST8PDT
Sub-patch  13923374; "Database Patch Set Update : 11.2.0.3.3 (13923374)"
Sub-patch  13696216; "Database Patch Set Update : 11.2.0.3.2 (13696216)"
Sub-patch  13343438; "Database Patch Set Update : 11.2.0.3.1 (13343438)"
   Bugs fixed:
     14480676, 13566938, 13419660, 10350832, 13632717, 14063281, 12919564
     13624984, 13430938, 13467683, 13588248, 13420224, 14548763, 13080778
     12646784, 13804294, 12861463, 12834027, 13377816, 13036331, 12880299
     14664355, 13499128, 14409183, 12998795, 12829021, 13492735, 12794305
     13503598, 10133521, 12718090, 13742433, 12905058, 12401111, 13742434
     13257247, 12849688, 13362079, 12950644, 13742435, 13464002, 12917230
     13923374, 12879027, 14613900, 12585543, 12535346, 14480675, 12588744
     11877623, 14480674, 13916709, 12847466, 13773133, 14076523, 13649031
     13340388, 13366202, 13528551, 13981051, 12894807, 13343438, 12582664
     12748240, 12797765, 13385346, 12923168, 13384182, 13612575, 13466801
     13484963, 12971775, 11063191, 13772618, 13070939, 12797420, 13035804
     13041324, 12976376, 11708510, 13742437, 13737746, 14062795, 13035360
     12693626, 13742438, 13326736, 13332439, 14038787, 14062796, 12913474
     13001379, 14390252, 13099577, 13370330, 13059165, 14062797, 14275605
     9873405, 13742436, 9858539, 14062794, 13358781, 12960925, 13699124
     12662040, 9703627, 12617123, 13338048, 12938841, 12658411, 12620823
     12845115, 12656535, 14062793, 12678920, 12764337, 13354082, 13397104
     14062792, 13250244, 12594032, 9761357, 12612118, 13742464, 13550185
     13457582, 13527323, 12780983, 12583611, 13502183, 12780098, 13705338
     13696216, 13476583, 11840910, 13903046, 13572659, 13718279, 13554409
     13657605, 13103913, 14063280
--------------------------------------------------------------------------------
OPatch succeeded.
[oracle@linora:/worktmp/14275605]$ $ORACLE_HOME/OPatch/opatch lsinventory -invPtrLoc \
/u01/app/oraInventory/oraInst.loc -bugs_fixed | egrep 'PSU|PATCH SET UPDATE'
14275605   14275605  Wed Mar 25 11:06:52 CST 2015   DATABASE PATCH SET UPDATE 11.2.0.3.4 (INCLUDES CPU
13923374   13923374  Wed Mar 25 11:06:44 CST 2015   DATABASE PATCH SET UPDATE 11.2.0.3.3 (INCLUDES 
13696216   13696216  Wed Mar 25 09:25:43 CST 2015   DATABASE PATCH SET UPDATE 11.2.0.3.2 (INCLUDES 
13343438   13343438  Wed Mar 25 09:17:40 CST 2015   DATABASE PATCH SET UPDATE 11.2.0.3.1
```
### 7.开启数据库及监听，注册PSU信息至数据库
```
SQL> alter system regeister;
SQL> @?/rdbms/admin/catbundle.sql psu apply
col action_time for a28
col version for a10
col comments for a35
col action for a25
col namespace for a12
select * from registry$history;
```
执行完catbundle.sql后，会在$ORACLE_HOME/rdbms/admin目录下生成
catbundle_PSU_<database SID>_APPLY.sql，catbundle_PSU_<database SID>_ROLLBACK.sql两个脚本
验证无误后开启其他数据库服务，如OGG、EM等。

### 8.回滚操作
关闭数据库相关服务，如监听、OGG、EM等。
```
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@linora:/worktmp/14275605]$ lsnrctl stop
```
opatch回滚patch 14275605
```
[oracle@linora:/worktmp/14275605]$ $ORACLE_HOME/OPatch/opatch rollback -id 14275605
```
执行完毕后，开启数据库，执行数据库级别的回滚操作：
```
SQL> alter system register;
SQL> @?/rdbms/admin/catbundle_PSU_LINORA_ROLLBACK.sql;
SQL> set line 200
SQL> col action_time for a28
SQL> col version for a10
SQL> col comments for a35
SQL> col action for a25
SQL> col namespace for a12
SQL> select * from registry$history;

ACTION_TIME                  ACTION         NAMESPACE    VERSION       ID COMMENTS              BUNDLE_SERIES
---------------------------- -------------- ------------ ---------- ----- --------------------- --------
17-SEP-11 10.21.11.595816 AM APPLY          SERVER       11.2.0.3       0 Patchset 11.2.0.2.0   PSU
24-MAR-15 02.33.46.001293 PM APPLY          SERVER       11.2.0.3       0 Patchset 11.2.0.2.0   PSU
25-MAR-15 09.20.32.232223 AM APPLY          SERVER       11.2.0.3       1 PSU 11.2.0.3.1        PSU
25-MAR-15 09.26.39.633258 AM APPLY          SERVER       11.2.0.3       2 PSU 11.2.0.3.2        PSU
25-MAR-15 10.14.32.978033 AM APPLY          SERVER       11.2.0.3       4 PSU 11.2.0.3.4        PSU
25-MAR-15 10.26.42.133461 AM ROLLBACK       SERVER       11.2.0.3       4 PSU 11.2.0.3.4        PSU

6 rows selected.
```

Reference:  
[Patching Oracle Software with OPatch](http://docs.oracle.com/cd/E11882_01/em.112/e12255/oui7_opatch.htm)

</br><b>EOF</b></br>
