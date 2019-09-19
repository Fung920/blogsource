---
title: RAC DG Broker ora-16843 ora-16839错误
categories: oracle
comments: false
date: 2019-09-07 17:24:50
tags: ['broker', 'dataguard']
---
新搭的一套ADG，在启用配置后，查询状态报以下错误：
```sql
DGMGRL> show configuration
Configuration - ADG_ZT

  Protection Mode: MaxAvailability
  Members:
  cdb01  - Primary database
    cdb02  - Physical standby database
    ztdb - Physical standby database
      Error: ORA-16843: errors discovered in diagnostic repository

DGMGRL> show database ztdb

Database - ztdb

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 0 seconds ago)
  Average Apply Rate: 29.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    ztdb1

  Database Error(s):
    ORA-16839: one or more user data files are missing

Database Status:
ERROR
```

一度怀疑restore的时候部分文件丢失了，对比主备库两边的数据文件，并没有异常，备库查询恢复/还原错误，也是正常的。
```sql
RMAN> report schema;
RMAN> list failure;
Database Role: PHYSICAL STANDBY
no failures found that match specification
```

MOS上有一个bug，怀疑是这个bug引起的：
[Bug 21495155 - data guard broker configuration shows ORA-16843 in RAC (Doc ID 21495155.8)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=16551594274140&id=21495155.8&displayIndex=1&_afrWindowMode=0&_adf.ctrl-state=i1oiz6ciq_81)

__Workaround__:
删除ADR下面的`HM_FINDING.ams`文件：
```sql
[oracle@ztdb diag]$ find ./ -name "metadata"
./rdbms/ztdb01/ztdb011/metadata
[oracle@ztdb diag]$ cd ./rdbms/ztdb01/ztdb011/metadata
[oracle@ztdb metadata]$ mv HM_FINDING.ams HM_FINDING.ams.old
DGMGRL> show  configuration

Configuration - ADG_ZT

  Protection Mode: MaxAvailability
  Members:
  cdb01  - Primary database
    cdb02  - Physical standby database
    ztdb01 - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 56 seconds ago)

[oracle@ztdb metadata]$ ls -ltr HM_FIND*
-rw-r----- 1 oracle oinstall 475136 9月   7 16:26 HM_FINDING.ams.old
-rw-r----- 1 oracle oinstall  65536 9月   7 17:15 HM_FINDING.ams
```

__EOF__
