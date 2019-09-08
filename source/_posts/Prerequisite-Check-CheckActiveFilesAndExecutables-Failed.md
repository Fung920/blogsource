---
title: Prerequisite Check "CheckActiveFilesAndExecutables" Failed
categories: oracle
comments: false
date: 2019-08-15 11:27:08
tags: patch
---
在对数据库进行打补丁的时候，遇到以下错误：
```sql
itjkdb01l:/oracle> opatch apply
Oracle Interim Patch Installer version 12.2.0.1.16
Copyright (c) 2019, Oracle Corporation.  All rights reserved.


Oracle Home       : /oracle/app/oracle/12.1.0
Central Inventory : /grid/app/oraInventory
   from           : /oracle/app/oracle/12.1.0/oraInst.loc
OPatch version    : 12.2.0.1.16
OUI version       : 12.1.0.2.0
Log file location : /oracle/app/oracle/12.1.0/cfgtoollogs/opatch/opatch2019-08-12_10-20-02AM_1.log

Verifying environment and performing prerequisite checks...
Prerequisite check "CheckActiveFilesAndExecutables" failed.
The details are:


Following active executables are not used by opatch process :
/oracle/app/oracle/12.1.0/bin/oracle

Following active executables are used by opatch process :

UtilSession failed: Prerequisite check "CheckActiveFilesAndExecutables" failed.
Log file location: /oracle/app/oracle/12.1.0/cfgtoollogs/opatch/opatch2019-08-12_10-20-02AM_1.log
OPatch failed with error code 73
```

* 原因
当使用opatch apply进行补丁应用时，在$ORACLE_HOME下的所有进程要被停止掉，如果有任何的进程在跑，就会报上述opatch 73的错误。

尝试用ps, lsof等命令查找出相关进程，如有必要，可以把它杀掉。
之后重启apply，恢复正常。

建议在执行apply前，运行以下命令:

```sql
$ORACLE_HOME/OPatch/opatch prereq CheckActiveFilesAndExecutables -ph ./
```


__EOF__

