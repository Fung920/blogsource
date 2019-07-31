---
title: OCR Voting disk replace or move on 11g and above
categories: oracle
comments: false
date: 2019-07-31 16:38:10
tags: how to
---
Essentially speaking, OCR disk is a two-way ASM disk, which can be replace via ASM disk rebalance method.
This post introduces another method by `ocrconfig` tool.
Assume we've created a new diskgroup name OCR_NEW to replace old diskgroup OCR. ___Pay attention to compatible.asm parameter of diskgroup.___
<!--more-->
## 1. Replace voting disk

```sql
--root user
crsctl replace votedisk +OCR_NEW
--check the result
crsctl query css votedisk
```

## 2. Replace OCR

```sql
--root user
ocrconfig -add +OCR_NEW
--check the result, the result should include +OCR_NEW and +OCR
ocrchek

--delete old diskgroup
ocrconfig -delete +OCR
```

## 3. Recreate spfile for ASM

* Check current spfile location

```sql
--grid user
$ORACLE_HOME/bin/gpnptool get -o- | xmllint --format - | grep -i spfile
asmcmd spget
```

* Recreate spfile to new diskgroup

```sql
--grid user
create pfile='/home/grid/asmpfile.ora' from spfile;
create spfile='+OCR_NEW' from pfile='/home/grid/asmpfile.ora';
```

## 4. Move password file to new diskgroup

```sql
--grid user
asmcmd pwmove --asm +OCR/orapwASM +OCR_NEW/orapwASM
```

## 5. Migrate mgmtdb(as of 12c)

MGMTDB is a built-in CDB database managed by Oracle GI, it stores Cluster Health Monitor information, by default, this database is localted on OCR disk. If we move OCR to another diskgroup, ensure this database also need to be moved.

## 5.1 Migrate by RMAN copy

* Check current status of mgmtdb

```sql
--grid user
srvctl status diskgroup -g ocr
srvctl config mgmtdb
srvctl status mgmtdb
```

* Restart mgmtdb to mount status

```sql
--grid user
srvctl stop mgmtdb
srvctl stop mgmtlsnr
```

* Backup mgmtdb

```sql
--grid user
export ORACLE_SID=-MGMTDB
RMAN> startup mount
RMAN> backup database format '/home/grid/backup/rman_mgmtdb_%U' tag='bk_db_move_dg';
```

* Restore spfile to new diskgroup

```sql
RMAN> restore spfile to "+OCR_NEW" from '/home/grid/backup';
```

Confirm the result:
```sql
srvctl config mgmtdb |grep Spfile
SQL> shutdown immediate
SQL> startup nomount
SQL> show parameter spfile
```

* Restore controlfile to new diskgroup

```sql
RMAN> shutdown immediate
RMAN> startup nomount
SQL> show parameter control_file
SQL> alter system set control_files='+OCR_NEW' scope=spfile ;
RMAN> restore controlfile from '/home/grid/backup/xxxxx';
```

* Migrate mgmtdb by rman copy

```sql
RMAN> alter database mount;
RMAN> backup as copy DEVICE TYPE DISK DATABASE FORMAT '+OCR_NEW';
RMAN> SWITCH DATABASE TO COPY;
--confirm the result, all tempfile should be resided in old diskgroup
RMAN> report schema;
```

* Recreate temp file

```sql
RMAN> run {
SET NEWNAME FOR TEMPFILE 1 TO '+OCR';
SET NEWNAME FOR TEMPFILE 2 TO '+OCR';
SET NEWNAME FOR TEMPFILE 3 TO '+OCR';
SWITCH TEMPFILE ALL;
}
```

* Verify migration result

```sql
RMAN> startup
RMAN> report schema;
--delete copy
RMAN> delete copy;
```

* Modify mgmtdb startup resource

```sql
--grid user
crsctl status res ora.mgmtdb -p |grep OCR

--replacing new diskgroup for start/stop dependency
crsctl modify res ora.mgmtdb -attr "START_DEPENDENCIES='hard(ora.MGMTLSNR,ora.OCR_NEW.dg) weak(uniform:ora.ons) pullup(ora.MGMTLSNR,ora.OCR_NEW.dg)'" -unsupported
crsctl modify res ora.mgmtdb -attr "STOP_DEPENDENCIES='hard(intermediate:ora.MGMTLSNR,intermediate:ora.asm,shutdown:ora.OCR_NEW.dg)'" -unsupported
```

The `unsupported` key word is for fixing below issue, because as 12c, Oracle does not support to modify ora* resource by crsctl, using srvctl to modify ora* resource instead, but didn't find anything about how to modify ora* resource by srvctl.

```sql
CRS-4995: The command ‘Modify resource’ is invalid in crsctl. Use srvctl for this command.
```

* Delete old diskgroup resource

```sql
--grid user, stop old diskgroup
srvctl stop diskgroup -g OCR
--restart gi, if nothing wrong, old diskgroup can be dropped
export ORACLE_SID=+ASM1
SQL> DROP DISKGROUP CRS INCLUDING CONTENTS;
```

## 5.2 Migrate by recreating

* Backup current configuration

```bash
[grid@racdb1:/home/grid]$ oclumon dumpnodeview -allnodes -v > ./mgmtdb.bak
```

* Stop and disable ora.crf resource on all nodes

```sql
[root@racdb1 ~]# /u01/app/12.1.0/grid/bin/crsctl stop res ora.crf -init
CRS-2673: Attempting to stop 'ora.crf' on 'racdb1'
CRS-2677: Stop of 'ora.crf' on 'racdb1' succeeded
[root@racdb2 ~]# /u01/app/12.1.0/grid/bin/crsctl stop res ora.crf -init
CRS-2673: Attempting to stop 'ora.crf' on 'racdb2'
CRS-2677: Stop of 'ora.crf' on 'racdb2' succeeded

--Disable ora.crf
[root@racdb1 ~]# /u01/app/12.1.0/grid/bin/crsctl modify res ora.crf -attr ENABLED=0 -init
[root@racdb2 ~]# /u01/app/12.1.0/grid/bin/crsctl modify res ora.crf -attr ENABLED=0 -init
```

* Delete mgmtdb by dbca

As grid user, locate the node mgmrdb is running:
```sql
srvctl status mgmtdb
```

As grid user, delete mgmtdb by dbca:

```sql
[grid@racdb1:/home/grid]$ dbca -silent -deleteDatabase -sourceDB -MGMTDB
Connecting to database
4% complete
9% complete
14% complete
19% complete
23% complete
28% complete
47% complete
Updating network configuration files
48% complete
52% complete
Deleting instance and datafiles
76% complete
100% complete
Look at the log file "/u01/app/grid/cfgtoollogs/dbca/_mgmtdb.log" for further details.
```

* Recreate mgmtdb

As grid user create container database:

```sql
[grid@racdb1:/home/grid]$ dbca -silent -createDatabase -sid -MGMTDB -createAsContainerDatabase true \
-templateName MGMTSeed_Database.dbc -gdbName _mgmtdb -storageType ASM \
-diskGroupName +CRS -datafileJarLocation $ORACLE_HOME/assistants/dbca/templates \
-characterset AL32UTF8 -autoGeneratePasswords -skipUserTemplateCheck

Registering database with Oracle Grid Infrastructure
5% complete
Copying database files
7% complete
9% complete
16% complete
23% complete
30% complete
37% complete
41% complete
Creating and starting Oracle instance
43% complete
48% complete
49% complete
50% complete
55% complete
60% complete
61% complete
64% complete
Completing Database Creation
68% complete
79% complete
89% complete
100% complete
Look at the log file "/u01/app/grid/cfgtoollogs/dbca/_mgmtdb/_mgmtdb0.log" for further details.
```

As grid user, create PDB, be aware, -pdbname is the cluster name, any '-' of cluster should be replaced by '_'
```sql
[grid@racdb1:/home/grid]$ dbca -silent -createPluggableDatabase -sourceDB -MGMTDB \
-pdbName racdb -createPDBFrom RMANBACKUP \
-PDBBackUpfile $ORACLE_HOME/assistants/dbca/templates/mgmtseed_pdb.dfb \
-PDBMetadataFile $ORACLE_HOME/assistants/dbca/templates/mgmtseed_pdb.xml \
-createAsClone true

Creating Pluggable Database
4% complete
12% complete
21% complete
38% complete
55% complete
85% complete
Completing Pluggable Database Creation
100% complete
Look at the log file "/u01/app/grid/cfgtoollogs/dbca/_mgmtdb/racdb/_mgmtdb.log" for further details.
```

* Post-creation

```sql
--check the status of mgmtdb with grid user
srvctl status MGMTDB

--secure the mgmtdb credential
mgmtca
```



* Start ora.crf on all nodes

```sql
--with root user on each node
[root@racdb1 ~]# /u01/app/12.1.0/grid/bin/crsctl modify res ora.crf -attr ENABLED=1 -init
[root@racdb2 ~]# /u01/app/12.1.0/grid/bin/crsctl modify res ora.crf -attr ENABLED=1 -init
[root@racdb1 ~]# /u01/app/12.1.0/grid/bin/crsctl start res ora.crf -init
CRS-2672: Attempting to start 'ora.crf' on 'racdb2'
CRS-2676: Start of 'ora.crf' on 'racdb2' succeeded
```



## 5.3 Migrate by MOS tool `mdbutil.pl`

Below example is migrating mgmtdb from OCR to CRS.

* Check the status

```bash
[grid@racdb1:/home/grid]$ ./mdbutil.pl --status
mdbutil.pl version : 1.98
2019-07-31 19:13:43: I Checking CHM status...
2019-07-31 19:14:06: I Listener MGMTLSNR is configured and running on racdb1
2019-07-31 19:14:10: I Database MGMTDB is configured and running on racdb1
2019-07-31 19:14:11: I Cluster Health Monitor (CHM) is configured and running
--------------------------------------------------------------------------------
CHM Repository Path = +OCR/_MGMTDB/8ECBF2858125261EE053C938A8C0A78B/DATAFILE/sysmgmtdata.259.1014914317
MGMTDB space used on DG +OCR = 4187 Mb
```

* Migrate mgmtdb files via mdbutil tool

```sh
[grid@racdb1:/home/grid]$ ./mdbutil.pl --mvmgmtdb --target=+CRS
mdbutil.pl version : 1.98
Moving MGMTDB, it will be stopped, are you sure (Y/N)? y
2019-07-31 19:15:57: I Checking for the required paths under +CRS
2019-07-31 19:16:06: I Creating new path +CRS/_MGMTDB/ONLINELOG
2019-07-31 19:16:08: I Creating new path +CRS/_MGMTDB/DATAFILE
2019-07-31 19:16:10: I Creating new path +CRS/_MGMTDB/DATAFILE/PDB$SEED
2019-07-31 19:16:10: I Creating new path +CRS/_MGMTDB/DATAFILE/TEMPFILE/PDB$SEED
2019-07-31 19:16:15: I Creating new path +CRS/_MGMTDB/DATAFILE/racdb_cluster
2019-07-31 19:16:16: I Creating new path +CRS/_MGMTDB/TEMPFILE/racdb_cluster
2019-07-31 19:16:17: I Getting MGMTDB Database files location
2019-07-31 19:16:17: I Getting MGMTDB Temp files location
2019-07-31 19:16:17: I Getting MGMTDB PDB PDB$SEED files location
2019-07-31 19:16:18: I Getting MGMTDB PDB PDB$SEED Temp files location
2019-07-31 19:16:18: I Getting MGMTDB PDB racdb_cluster files location
2019-07-31 19:16:19: I Getting MGMTDB PDB racdb_cluster Temp files location
2019-07-31 19:16:23: I Creating temporary PFILE
2019-07-31 19:16:23: I Creating target SPFILE
2019-07-31 19:16:28: I Stopping mgmtdb
2019-07-31 19:16:51: I Copying MGMTDB DBFiles to +CRS
2019-07-31 19:17:00: I Copying MGMTDB PDB$SEED DBFiles to +CRS
2019-07-31 19:17:04: I Copying MGMTDB PDB DBFiles to +CRS
2019-07-31 19:17:23: I Creating the CTRL File
2019-07-31 19:19:03: I The CTRL File has been created and MGMTDB is now running from +CRS
2019-07-31 19:19:03: I Setting MGMTDB SPFile location
2019-07-31 19:19:07: I Modifing the init parameter
2019-07-31 19:19:07: I Removing old MGMTDB
2019-07-31 19:19:20: I Changing START_DEPENDENCIES
2019-07-31 19:19:21: I Changing STOP_DEPENDENCIES
2019-07-31 19:19:22: I Restarting MGMTDB using target SPFile
2019-07-31 19:21:12: I MGMTDB Successfully moved to +CRS!
```

* Check the result

```sh
[grid@racdb1:/home/grid]$ ./mdbutil.pl --status
mdbutil.pl version : 1.98
2019-07-31 19:23:18: I Checking CHM status...
2019-07-31 19:23:20: I Listener MGMTLSNR is configured and running on racdb1
2019-07-31 19:23:21: I Database MGMTDB is configured and running on racdb1
2019-07-31 19:23:21: I Cluster Health Monitor (CHM) is configured and running
--------------------------------------------------------------------------------
CHM Repository Path = +CRS/_MGMTDB/DATAFILE/racdb_cluster/sysmgmtdata.20190731191620.dbf
MGMTDB space used on DG +CRS = 742 Mb
--------------------------------------------------------------------------------
```


Reference:
[How to Move/Recreate GI Management Repository to Different Shared Storage (Diskgroup, CFS or NFS etc) (Doc ID 1589394.1)](https://support.    oracle.com/epmos/faces/DocumentDisplay?_afrLoop=150771986253294&id=1589394.1&_adf.ctrl-state=163b3koqeg_61)
[crsctl modify ora.* resource fails with CRS-4995 in 12.1.0.2 and above (Doc ID 1918102.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=150867291521193&id=1918102.1&_adf.ctrl-state=163b3koqeg_175)
[MDBUtil: GI Management Repository configuration tool (Doc ID 2065175.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=154466631555105&id=2065175.1&_adf.ctrl-state=163b3koqeg_290#aref_section31)

__EOF__

