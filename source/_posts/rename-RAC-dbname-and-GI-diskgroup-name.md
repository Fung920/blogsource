---
title: Rename RAC dbname and GI diskgroup name
categories: oracle
comments: false
date: 2019-02-21 16:40:11
tags: RAC
---

Client wants to use Oracle OEM to manage all the database RACs, due to unoptimized plan of installation, they want to standardize naming conversion of DBNAME and diskgroup name, with `NID` utility, it's easy to modify the database name, but modify diskgroup name is a bit little complicated.

<!--more-->
# 1. Modify DBNAME with nid
First of all, backup is essential before any changes.

* Bring RAC database in exclusive mode

```sql
SQL> alter system set cluster_database=false scope=both;
```

* Stop the database and bring database in mount mode

```sql
$srvctl stop database -d orcl
$sqlplus "/as sysdba"
SQL> startup mount
```

* Modify dbname with nid

```sql
<localhost:/home/oracle>$ nid target=/ dbname=ccms
DBNEWID: Release 12.1.0.2.0 - Production on Thu Feb 21 11:59:50 2019
Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.
Connected to database ORCL (DBID=625082333)
Connected to server version 12.1.0
Control Files in database:
    +DATA/ORCL/CONTROLFILE/current.267.998418721
Change database ID and database name ORCL to CCMS? (Y/[N]) => y
Proceeding with operation
Changing database ID from 625082333 to 2262823414
Changing database name from ORCL to CCMS
    Control File +DATA/ORCL/CONTROLFILE/current.267.998418721 - modified
    Datafile +DATA/ORCL/DATAFILE/system.270.99841872 - dbid changed, wrote new name
    Datafile +DATA/ORCL/DATAFILE/sysaux.271.99841872 - dbid changed, wrote new name
    Datafile +DATA/ORCL/DATAFILE/undotbs1.272.99841872 - dbid changed, wrote new name
    Datafile +DATA/ORCL/DATAFILE/undotbs2.274.99841874 - dbid changed, wrote new name
    Datafile +DATA/ORCL/DATAFILE/users.275.99841874 - dbid changed, wrote new name
    Datafile +DATA/ORCL/TEMPFILE/temp.273.99841872 - dbid changed, wrote new name
    Control File +DATA/ORCL/CONTROLFILE/current.267.998418721 - dbid changed, wrote new name
    Instance shut down
Database name changed to CCMS.
Modify parameter file and generate a new password file before restarting.
Database ID for database CCMS changed to 2262823414.
All previous backups and archived redo logs for this database are unusable.
Database has been shutdown, open database with RESETLOGS option.
Succesfully changed database name and ID.
DBNEWID - Completed succesfully.
```

* Bring database back in mount mode and modify DBNAME in database

```sql
SQL> startup mount
ORACLE instance started.

Total System Global Area 3.2481E+11 bytes
Fixed Size		    7668016 bytes
Variable Size		 4.6171E+10 bytes
Database Buffers	 2.7703E+11 bytes
Redo Buffers		 1602940928 bytes
ORA-01103: database name 'CCMS' in control file is not 'ORCL'

SQL> alter system set db_name=ccms scope=spfile;
System altered.
--Bring database backup in cluster mode
SQL> alter system set cluster_database=true scope=spfile;
System altered.
SQL> startup force mount
```

* Open database with resetlogs option

```sql
SQL> alter database open resetlogs;
Database altered.
SQL> show parameter name
NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name		     string
db_file_name_convert		     string
db_name 			     string	 CCMS
db_unique_name			     string	 CCMS
global_names			     boolean	 FALSE
instance_name			     string	 orcl1
lock_name_space 		     string
log_file_name_convert		     string
pdb_file_name_convert		     string
processor_group_name		     string
service_names			     string	 CCMS
```

* Modify instance names

```sql
SQL> create pfile='/home/oracle/pfile.ora' from spfile;
--replace old instance name with new instance in pfile
--It's recommened create and initSID.ora file in $ORACLE_HOME/dbs
SQL> startup nomount
SQL> create spfile='+DATA/ORCL/spfile.ora' from pfile='/home/oracle/pfile.ora';
```

* Remove old database and register new database

```sql
SQL> shutdown immediate
$srvctl remove database -d orcl
$srvctl add database -d ccms -o $ORACLE_HOME -spfile '+DATA/ORCL/spfile.ora' -startoption OPEN -policy AUTOMATIC -v
--add instance attach to GI
$srvctl add instance -d ccms -i ccms1 -n localhost1
$srvctl add instance -d ccms -i ccms2 -n localhost2
--check the status of database
$srvctl config database -d ccms
$crsctl stat res -t
SQL> show parameter name

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name		     string
db_file_name_convert		     string
db_name 			     string	 ccms
db_unique_name			     string	 ccms
global_names			     boolean	 FALSE
instance_name			     string	 ccms1
lock_name_space 		     string
log_file_name_convert		     string
pdb_file_name_convert		     string
processor_group_name		     string
service_names			     string	 ccms
```

# 2. Modify ASM diskgroup name with `renamedg`

* Dismount diskgroups which need to be renamed

```sql
$srvctl stop database -d ccms
--with grid user dismount diskgroup with both node
</home/grid>$ sqlplus / as sysasm
SQL> alter diskgroup DATA dismount;
```

* Rename DG name with `renamedg`

```sql
</home/grid>$ renamedg phase=both dgname=DATA \
newdgname=DATADG asm_diskstring='/dev/asmdisk*' verbose=true
Parsing parameters..

Parameters in effect:

 	 Old DG name       : DATA
	 New DG name          : DATADG
	 Phases               :
	 	 Phase 1
	 	 Phase 2
	 Discovery str        : /dev/asmdisk*
	 Clean              : TRUE
	 Raw only           : TRUE
renamedg operation: phase=both dgname=DATA newdgname=DATADG asm_diskstring=/dev/asmdisk* verbose=true
Executing phase 1
Discovering the group
Performing discovery with string:/dev/asmdisk*
Identified disk UFS:/dev/asmdisk5 with disk number:1 and timestamp (33081105 1165527040)
Identified disk UFS:/dev/asmdisk4 with disk number:0 and timestamp (33081105 1165527040)
Checking for hearbeat...
Re-discovering the group
Performing discovery with string:/dev/asmdisk*
Identified disk UFS:/dev/asmdisk5 with disk number:1 and timestamp (33081105 1165527040)
Identified disk UFS:/dev/asmdisk4 with disk number:0 and timestamp (33081105 1165527040)
Checking if the diskgroup is mounted or used by CSS
Checking disk number:1
Checking disk number:0
Generating configuration file..

ccms1.__large_pool_size=8053063680
Completed phase 1

ccms1.__data_transfer_cache_size=0
Executing phase 2
Looking for /dev/asmdisk5
Modifying the header
Looking for /dev/asmdisk4
Modifying the header
Completed phase 2
Terminating kgfd context 0x7f09cb8ec0a0
```

* Remove old diskgroup name in GI

```sql
SQL> alter diskgroup DATADG mount ;
$srvctl modify database -db ccms -o $ORACLE_HOME -spfile '+DATADG/ORCL/spfile.ora' -diskgroup 'DATADG'
$srvctl remove diskgroup -g DATA
```
* Re-create controlfile to reflect new datafile localtion

```sql
SQL> alter database backup controlfile to trace as '/home/oracle/cntl.ctl';
--modify cntl.ctl to reflect new file locations
SQL> STARTUP NOMOUNT
CREATE CONTROLFILE REUSE DATABASE "CCMS" RESETLOGS  NOARCHIVELOG
    MAXLOGFILES 192
    MAXLOGMEMBERS 3
    MAXDATAFILES 1024
    MAXINSTANCES 32
    MAXLOGHISTORY 2920
LOGFILE
  GROUP 1 '+DATADG/CCMS/ONLINELOG/group_1.268.1000814621'  SIZE 50M BLOCKSIZE 512,
  GROUP 2 '+DATADG/CCMS/ONLINELOG/group_2.269.1000814621'  SIZE 50M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '+DATADG/ORCL/DATAFILE/system.270.998418723',
  '+DATADG/ORCL/DATAFILE/sysaux.271.998418725',
  '+DATADG/ORCL/DATAFILE/undotbs1.272.998418727',
  '+DATADG/ORCL/DATAFILE/undotbs2.274.998418741',
  '+DATADG/ORCL/DATAFILE/users.275.998418741'
CHARACTER SET AL32UTF8
;
--open database with resetlogs option
SQL> alter database open resetlogs;
```

* Modify pfile to reflect new controlfile location

Don't forget to modify `cluster_database=true` and set `db_create_file_dest` to new diskgroup name.
```sql
SQL> create spfile='+DATADG/ORCL/spfile.ora' from pfile='/home/oracle/pfile.ora'
$srvctl stop database -d ccms
$srvctl start database -d ccms
```

# 3. Troubleshooting & Tips
* Open resetlogs encountered ORA-01618

```sql
--Enable log thread 2 in node 1
SQL> alter database enable thread 2;
--Create log file in node 2
ALTER DATABASE ADD LOGFILE THREAD 2
  GROUP 9 ('+DATADG') SIZE 4096M BLOCKSIZE 512,
  GROUP 10 ('+DATADG') SIZE 4096M BLOCKSIZE 512,
  GROUP 11 ('+DATADG') SIZE 4096M BLOCKSIZE 512,
  GROUP 12 ('+DATADG') SIZE 4096M BLOCKSIZE 512;
```

Before using `renamedg`, all the diskgroups must be dismounted from all nodes, and it's recommended to specify `ask_disktrings` parameter in the command line. Otherwise, below error maybe occur:
```
KFNDG-00407: Could not find disks for disk group
```



__EOF__
