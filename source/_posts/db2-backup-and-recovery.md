---
layout: post
title: "DB2 Backup and Recovery"
date: 2017-08-22 02:07:44
comments: false
categories: db2
tags: backup
keywords: db2, backup, recovery
description:
---
Backup always essential. This post will show how to backup and recover a DB2 database.
<!--more-->

# 1. Offline backup

Keep db2 instance running and deactivate the databases:

```
[db2inst1@db2srv ~]$ /worktmp/fptool.ksh -d
+++Forcing applications and deactivating databases TESTDB...
+++
DB20000I  The FORCE APPLICATION command completed successfully.
DB21024I  This command is asynchronous and may not be effective immediately.
DB20000I  The DEACTIVATE DATABASE command completed successfully.

[db2inst1@db2srv ~]$ db2 backup db testdb to /db2backup/ PARALLELISM 4 compress
Backup successful. The timestamp for this backup image is : 20170822140451
```

Use below command to figure out the backup progress:

```
[db2inst1@db2srv ~]$ db2 list utilities show detail
ID                               = 3
Type                             = BACKUP
Database Name                    = TESTDB
Member Number                    = 0
Description                      = offline db
Start Time                       = 08/22/2017 14:04:50.296195
State                            = Executing
Invocation Type                  = User
Throttling:
   Priority                      = Unthrottled
Progress Monitoring:
   Estimated Percentage Complete = 20
      Total Work                 = 160078696 bytes
      Completed Work             = 32618320 bytes
      Start Time                 = 08/22/2017 14:04:50.296209
```

# 2. Online backup
Online backup always a little bit more complicated then offline backup, because whenever refer to the "online", it means the database must enable archive log mode. Besides, the TRACKMOD must be enabled if the incremental backup is needed.

## 2.1 Enabling the archive log on DB2

Below error means the database aren't in archive log mode.

```
[db2inst1@db2srv ~]$ db2 activate db testdb
DB20000I  The ACTIVATE DATABASE command completed successfully.
[db2inst1@db2srv ~]$ db2 backup db testdb online to /db2backup include logs
SQL2413N  Online backup is not allowed because the database is not recoverable
or a backup pending condition is in effect.
```

Enabling the archive log mode:

```
[db2inst1@db2srv ~]$ db2 update db cfg for testdb using LOGARCHMETH1 disk:/db2/archive/
```

## 2.2 Enabling the TRACKMOD

```
[db2inst1@db2srv ~]$ db2 get db cfg for testdb |grep -i track
 Track modified pages                         (TRACKMOD) = NO
[db2inst1@db2srv ~]$ db2 update db cfg for testdb using trackmod yes immediate
```

## 2.3 Take an online backup/incremental backup

Take a offline backup before the online backup.

```
[db2inst1@db2srv ~]$ db2 deactivate db testdb
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2inst1@db2srv ~]$ db2 activate db testdb
SQL1116N  A connection to or activation of database "TESTDB" failed because
the database is in BACKUP PENDING state.  SQLSTATE=57019
[db2inst1@db2srv ~]$ db2 backup db testdb to /dev/null

Backup successful. The timestamp for this backup image is : 20170822144259
```

For the online backup and incremental backup:

```
[db2inst1@db2srv ~]$ db2 activate db testdb
DB20000I  The ACTIVATE DATABASE command completed successfully.
[db2inst1@db2srv ~]$ db2 backup db testdb online to /db2backup/ include logs
Backup successful. The timestamp for this backup image is : 20170822144415

[db2inst1@db2srv ~]$ db2 backup db testdb online incremental to /db2backup/ include logs
Backup successful. The timestamp for this backup image is : 20170822144533
```

## 2.4 Verification of the backup image

DB2 provide <code>db2ckbkp</code> to check the integrity of the backup images.

```
[db2inst1@db2srv db2backup]$ db2ckbkp TESTDB.0.db2inst1.DBPART000.20170822140451.001
[1] Buffers processed:    #############
Image Verification Complete - successful.
```

To check the backup image file header:

```
[db2inst1@db2srv db2backup]$ db2ckbkp -h TESTDB.0.db2inst1.DBPART000.20170822140451.001


=====================
MEDIA HEADER REACHED:
=====================
    Server Database Name           -- TESTDB
    Server Database Alias          -- TESTDB
    Client Database Alias          -- TESTDB
    Timestamp                      -- 20170822140451
    Database Partition Number      -- 0
    Instance                       -- db2inst1
    Database Configuration Type    -- 0 (Non-shared data)
    Sequence Number                -- 1
    Database Member ID             -- 0
    Release ID                     -- 0x1400 (DB2 v11.1.2.2)
    AL version                     -- V:11 R:1 M:2 F:2 I:0 SB:0
    Database Seed                  -- 0x6ABD4EE0
    DB Comment's Codepage (Volume) -- 0
    DB Comment (Volume)            --
    DB Comment's Codepage (System) -- 0
    DB Comment (System)            --
    Authentication Value           -- 255 (Not specified)
    Backup Mode                    -- 0 (Offline)
    Includes Logs                  -- 0 (No)
    Compression                    -- 1 (Compressed)
    Backup Type                    -- 0 (Database-level)
    Backup Granularity             -- 0 (Non-incremental)
    Merged Backup Image            -- 0 (No)
    Status Flags                   -- 0x1
                                      Consistent on this member
    System Catalogs in this image  -- 1 (Yes)
    Catalog Partition Number       -- 0
    DB Codeset                     -- UTF-8
    DB Territory                   -- US
    LogID                          -- 1503316320
    LogPath                        -- /db2/db2inst1/NODE0000/SQL00001/LOGSTREAM0000/
    Backup Buffer Size             -- 2101248 (513 4K pages)
    Number of Sessions             -- 1
    Platform                       -- 0x1E (Linux-x86-64)
    Encrypt Info Flags             -- 0x0

 The proper image file name would be:
TESTDB.0.db2inst1.DBPART000.20170822140451.001

[1] Buffers processed:    #############
Image Verification Complete - successful.
```

To check the logs needed for roll forward:

```
[db2inst1@db2srv db2backup]$ db2ckbkp -a TESTDB.0.db2inst1.DBPART000.20170822144533.001 |grep -i "File Number"
                      File Number   [000] = 1
                      File Number   [001] = 2
                      File Number   [002] = 3
```

Display the LFH (Log File Header) and MFH (Mirror LFH) data:

```
[db2inst1@db2srv db2backup]$ db2ckbkp -l TESTDB.0.db2inst1.DBPART000.20170822144533.001 |grep -i lsn
          Database activation LSN = 0000000000040391
          ...
                  Min LSN To Undo = 0000000000000000
          ....
                       lowtranlsn = 0000000000040393
                       minbufflsn = 0000000000040393
                  groupMinBuffLSN = 0000000000040393
                          headlsn = 0000000000040393
                     groupHeadLsn = 0000000000040393
    initialRecoveryStartingLFSLSN = 0/0000000000000000
                       startupLsn = 0000000000040391
                         myRegLsn = 0000000000000000
```

Use <code>db2flsn</code> to figure out the log sequence according to above result:

```bash
[db2inst1@db2srv db2backup]$ db2flsn -db testdb 0000000000040393
Given LSN is in log file S0000001.LOG
```

# 3. Disaster recovery

## 3.1 Incomplete restore

```
[db2inst1@db2srv backup]$ date
Fri Aug  7 10:59:14 CST 2015
[db2inst1@db2srv backup]$
[db2inst1@db2srv backup]$ db2 "delete from (select * from t where id='51369')"
DB20000I  The SQL command completed successfully.
[db2inst1@db2srv backup]$ db2 commit
DB20000I  The SQL command completed successfully.
[db2inst1@db2srv backup]$ db2 restore db testdb incremental automatic from /data2/backup/ taken at 20150807105110
[db2inst1@db2srv backup]$ db2 rollforward db testdb to 2015-08-07.10.57.41.00000 using local time
[db2inst1@db2srv backup]$ db2 rollforward db testdb stop
[db2inst1@db2srv backup]$ db2 "select * from t"
ID          NAME
----------- --------------------
      51369 Fung
```

## 3.2 Table space level restore

```
[db2inst1@db2srv backup]$ db2 "restore db testdb tablespace (FUNG) online from /data2/backup taken at 20150807101213"
[db2inst1@db2srv backup]$ db2 "rollforward db testdb to end of logs tablespace (FUNG)"
```

## 3.3 Incremental recovery
Finding which backup set should be used for incremental recovery:

```
[db2inst1@db2srv db2backup]$ db2ckrst -d testdb -t 20170822144533

Suggested restore order of images using timestamp 20170822144533 for
database testdb.
====================================================================
 restore db testdb incremental taken at 20170822144533
 restore db testdb incremental taken at 20170822144415
 restore db testdb incremental taken at 20170822144533
====================================================================
```

Example for incremental restore:

```
[db2inst1@db2srv db2backup]$ db2 drop db testdb
DB20000I  The DROP DATABASE command completed successfully.
[db2inst1@db2srv db2backup]$ db2 restore db testdb incremental automatic taken at 20170822144533
DB20000I  The RESTORE DATABASE command completed successfully.
[db2inst1@db2srv db2backup]$ db2 rollforward db testdb to end of logs and stop
```

## 3.4 Restoring the logs

For the include log backup, user can only restore the logs, and roll forward using these logs:

```
[db2inst1@db2srv backup]$ db2 backup db testdb online to /data2/backup include logs
Backup successful. The timestamp for this backup image is : 20150807145504

[db2inst1@db2srv backup]$ db2 restore db testdb logs from /data2/backup/ taken at 20150807145504 logtarget /data2/logs/
DB20000I  The RESTORE DATABASE command completed successfully.
[db2inst1@db2srv backup]$ ll /data2/logs/
total 12
-rw-------. 1 db2inst1 db2iadm1 12288 Aug  7 15:02 S0000022.LOG
```

If restore with the database, the LOGS keyword not necessary:
```
db2 restore db testdb from /data2/backup/ taken at 20150807145504 logtarget /data2/logs/
db2 "rollforward db testdb to end of logs and stop overflow log path (/data2/logs) "
```
**The overflow path is db2 looking for logs where rollforward needed**

## 3.5 Restore to a different machine
While performing migrating to another machine, there are three parameters which indicates where the database path are:

* TO target_directory
* DBPATH ON target_directory
* ON path_list

If the target database doesn't exist, the <code>TO</code> and <code>DBPATH ON</code> keyword will specify the target database's catalog, <code>ON</code> specify the AutoStorage path, and database catalog will be placed in the first directory where <code>ON</code> specify.

```bash
db2 restore db testdb into testdb taken at TIMESTAMP to /data
db2 restore db testdb into testdb taken at TIMESTAMP DBPATH ON /data
db2 restore db testdb into testdb taken at TIMESTAMP on '/data', '/data2'
```

## 3.6 Redirect restore
Redirect option can modify the container directory(except Auto Storage), you can generate a redirect script and modify it to meet your business needed.

```
[db2inst1@db2srv db2backup]$ db2 restore db testdb taken at 20170822140451 redirect generate script redirect.dll
db2 -tvf redirect.dll
```

## 3.7 Roll forward usages

**query status**           : query current database status, to see if in rollforward pending.    

**stop/complate**          : stop the rolling forward of log records, and completes the rollforward recovery process by rolling back incomplete transactions and turning off the rollforward pending state of the database.   

**cancel**                 : cancels the rollforward recovery operation. This puts the database in restore pending state.  

**ONLINE**                 : Tablespace level recovery to be done online. Makes the database can be accessible while rollforward in progress.


```bash
[db2inst1@db2srv db2backup]$ db2 rollforward db testdb query status
                                 Rollforward Status
 Input database alias                   = testdb
 Number of members have returned status = 1

 Member ID                              = 0
 Rollforward status                     = not pending
 Next log file to be read               =
 Log files processed                    = S0000001.LOG - S0000001.LOG
 Last committed transaction             = 2017-08-22-07.14.14.000000 UTC
```

***EOF***
