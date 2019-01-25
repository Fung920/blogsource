---
title: Issues of building RAC ADG
categories: oracle
comments: false
date: 2019-01-11 12:45:18
tags:
---
The background of this case is that our customer want to build a RAC to RAC ADG, we used duplicate database from backup set to restore the ADG, but because the source database is very huge, about 16TB, all the backup sets were stored in NFS file system which shared by primary and standby. Customer's network bandwidth is 1000Mb, but the transportation speed is under 30MB/s, it took a very long time to restore the whole database.

# 1. Missing directory
The first problem we encountered is data file name non-normalization, because the ASM diskgroups are the same with both primary and standby database, we didn't specify the set new name command, almost all the datafiles in this backup piece were failed to be restored.But it didn't impact the whole restore progress, duplicate process would restore those missing datafiles from previous backupset, or recreate them with empty contents.
```sql
--error messages because of datafile name's non-normalization
channel ORA_AUX_DISK_1: restoring datafile 00555 to +dgdata5
channel ORA_AUX_DISK_1: reading from backup piece /nfs_file/fullbakcup_4htlbj6l_1_1
channel ORA_AUX_DISK_1: ORA-19870: error while restoring backup piece /nfs_file/fullbakcup_4htlbj6l_1_1
--we didn't have +DGDATA3/proddb directory
ORA-19504: failed to create file "+DGDATA3/proddb/datafile/tbscrj_data_1_647.dbf"
ORA-17502: ksfdcre:4 Failed to create file +DGDATA3/proddb/datafile/tbscrj_data_1_647.dbf
ORA-15173: entry 'proddb' does not exist in directory '/'
...

--duplicate process tring to restore from previous backup
channel ORA_AUX_DISK_1: reading from backup piece /nfs_file/fullbakcup_4ptlgovj_1_1
channel ORA_AUX_DISK_1: piece handle=/nfs_file/fullbakcup_4ptlgovj_1_1 tag=TAG20181221T175217
channel ORA_AUX_DISK_1: restored backup piece 1
channel ORA_AUX_DISK_1: restore complete, elapsed time: 20:44:49
failover to previous backup

--re-creating missing datafile
creating datafile file number=331 name=+DGDATA3/proddb/datafile/tbscrj_data_1_647.dbf
creating datafile file number=340 name=+dgdata3

--because previous backup recored in control files are years ago, the backup set can't be found now
channel ORA_AUX_DISK_1: ORA-19870: error while restoring backup piece /backup/DB_PRODDB_s70_p1_t939760538
ORA-19505: failed to identify file "/backup/DB_PRODDB_s70_p1_t939760538"
ORA-27037: unable to obtain file status
IBM AIX RISC System/6000 Error: 2: No such file or directory
Additional information: 3
```
After restore finished, duplicate process would try to recover the standby, but the archived logs which need to be applied by recover process are years ago, recover process was failed again. All we need to do is restore those impacted files and recover them.

## 1.1 resolution
One of the resolution is restore the missing datafiles from backup set, or take data file copy from primary:

* restore from backup set

Every time you issue a restore command, rman would read relevant backup piece once, it's recommended to use one restore command to restore multiple datafiles, especial your backup piece is very large:
```sql
rman target / <<EOF
run {
allocate channel c1 type disk;
allocate channel c2 type disk;
restore datafile 340, 331, 342;
release channel c1;
release channel c2;
}
EOF
```

* restore from datafile copy

Take the datafile copy from primary:
```sql
RMAN>  backup as copy datafile 319 format '/arch2/data319.dbf';
Starting backup at 2019-01-04 10:01:24
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=387 instance=proddb2 device type=DISK
channel ORA_DISK_1: starting datafile copy
input datafile file number=00319 name=+DGDATA3/proddb/datafile/tbscrj_fq_dzd_2017.382.940007677
output file name=/arch2/data319.dbf tag=TAG20190104T100125 RECID=4 STAMP=996660397
channel ORA_DISK_1: datafile copy complete, elapsed time: 00:05:15
Finished backup at 2019-01-04 10:06:40
```

scp to the standby and catalog with it:
```sql
RMAN> catalog datafilecopy '/gglog/data319.dbf';

  using target database control file instead of recovery catalog
  cataloged datafile copy
  datafile copy file name=/gglog/data319.dbf RECID=1085 STAMP=996663809

RMAN> list copy of datafile 319;

  List of Datafile Copies
  =======================

  Key     File S Completion Time     Ckp SCN    Ckp Time
  ------- ---- - ------------------- ---------- -------------------
  1085    319  A 2019-01-04 11:03:29 16326666095593 2019-01-04 10:01:25
          Name: /gglog/data319.dbf
          Tag: TAG20190104T100125
RMAN> restore datafile 319;
```

# 2. Backup from standby side
If you want to backup in standby side, you must resync catalog from primary, otherwise, it will get the below errors:
```sql
resyncing from database with DB_UNIQUE_NAME PRODDB
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03009: failure of resync command on default channel at 11/22/2016 10:49:00
ORA-17629: Cannot connect to the remote database server
ORA-17627: ORA-00942: table or view does not exist
```

* resync catalog from primary

```sql
RMAN> connect catalog rman/<rman password>@RMAN_REMOTE_CAT
connected to recovery catalog database

RMAN>  show all for db_unique_name PRODDB;
```

* resync catalog from standby

```sql
RMAN> resync catalog;
```
After that, run the usual backup script in the standby.



<!--more-->


__EOF__
