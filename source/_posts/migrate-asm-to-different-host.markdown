---
layout: post
title: "Migrate ASM to different host"
date: 2015-06-14 10:46:44
comments: false
categories: oracle
tags: 
keywords: migration
description: ASM数据库主机更换存储及主机
---
客户搬迁机房，同时更换存储和主机，这是一台单实例的11g ASM数据库，通过底层存储镜像进行数据的迁移，在迁移过程中，还是遇到了一些小问题，在此记录一下。
<!--more-->
接手时的环境是客户已经自行安装好GI及数据库软件，ASM的diskgroup只有一个DATA盘，因此，只需要将部分资源添加进GI管理即可，初始GI状态如下：
```
[grid@orl6:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
NAME           TARGET  STATE        SERVER                   STATE_DETAILS       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.ons
               OFFLINE OFFLINE      orl6                                         
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        OFFLINE OFFLINE                                                   
ora.diskmon
      1        OFFLINE OFFLINE                                                   
ora.evmd
      1        ONLINE  ONLINE       orl6                                         
```
### 1、修改cssd资源为自启动：
```
[grid@orl6:/home/grid]$ crs_stat -p ora.cssd
NAME=ora.cssd
TYPE=ora.cssd.type
ACTION_SCRIPT=
ACTIVE_PLACEMENT=0
AUTO_START=never
CHECK_INTERVAL=30
DESCRIPTION="Resource type for CSSD"
FAILOVER_DELAY=0
FAILURE_INTERVAL=3
FAILURE_THRESHOLD=5
HOSTING_MEMBERS=
PLACEMENT=balanced
RESTART_ATTEMPTS=5
SCRIPT_TIMEOUT=600
START_TIMEOUT=600
STOP_TIMEOUT=900
UPTIME_THRESHOLD=1m

[grid@orl6:/home/grid]$ crsctl modify resource "ora.cssd" -attr "AUTO_START=1" 
```
### 2、使用netca创建监听
重启HA，修改后GI状态如下：
```
[grid@orl6:/home/grid]$ crsctl stop has && crsctl start has
[grid@orl6:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
NAME           TARGET  STATE        SERVER                   STATE_DETAILS       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       orl6                                         
ora.ons
               OFFLINE OFFLINE      orl6                                         
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       orl6                                         
ora.diskmon
      1        OFFLINE OFFLINE                                                   
ora.evmd
      1        ONLINE  ONLINE       orl6                                         
```
### 3、添加ASM资源到GI
```
[grid@orl6:/home/grid]$ srvctl add asm -d /dev/asm*
[grid@orl6:/home/grid]$ crsctl modify resource "ora.asm" -attr "AUTO_START=1"
[grid@orl6:/home/grid]$ crsctl start res ora.asm
CRS-2672: Attempting to start 'ora.asm' on 'orl6'
CRS-2676: Start of 'ora.asm' on 'orl6' succeeded
[grid@orl6:/home/grid]$ 
[grid@orl6:/home/grid]$ sqlplus / as sysasm
SQL> alter diskgroup DATA1 mount;

Diskgroup altered.
[grid@orl6:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
NAME           TARGET  STATE        SERVER                   STATE_DETAILS       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA1.dg
               ONLINE  ONLINE       orl6                                         
ora.LISTENER.lsnr
               ONLINE  ONLINE       orl6                                         
ora.asm
               ONLINE  ONLINE       orl6                     Started             
ora.ons
               OFFLINE OFFLINE      orl6                                         
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       orl6                                         
ora.diskmon
      1        OFFLINE OFFLINE                                                   
ora.evmd
      1        ONLINE  ONLINE       orl6                                         
```
### 4、添加database资源到GI
添加前，先把旧机器上listener.ora等文件copy过来，同时要保证相关目录存在，添加spfile位置等，在客户现场遇到的问题就是GI在起的过程中，local_listener报错，其设置为tnsnames解析，将旧设备上tnsnames.ora复制过来即可。
```
[oracle@orl6:/home/oracle]$ vi $ORACLE_HOME/dbs/initkyun.ora
SPFILE='+DATA1/kyun/spfilekyun.ora'
[oracle@orl6:/home/oracle]$ mkdir -p $ORACLE_BASE/admin/$ORACLE_SID/adump
[oracle@orl6:/home/oracle]$ srvctl add database -d kyun -o /u01/app/oracle/product/11gr2
[oracle@orl6:/home/oracle]$ 
[grid@orl6:/home/grid]$ srvctl start database -d kyun
[grid@orl6:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
NAME           TARGET  STATE        SERVER                   STATE_DETAILS       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA1.dg
               ONLINE  ONLINE       orl6                                         
ora.LISTENER.lsnr
               ONLINE  ONLINE       orl6                                         
ora.asm
               ONLINE  ONLINE       orl6                     Started             
ora.ons
               OFFLINE OFFLINE      orl6                                         
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       orl6                                         
ora.diskmon
      1        OFFLINE OFFLINE                                                   
ora.evmd
      1        ONLINE  ONLINE       orl6                                         
ora.kyun.db
      1        ONLINE  ONLINE       orl6                     Open                
```
最后，重启集群和主机，看看数据库是否会随HA启动

</br>
<b>EOF</b>
</br>

