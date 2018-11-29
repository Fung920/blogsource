---
layout: post
title: "Part of 12c New Features"
date: 2016-03-31 15:31:05
comments: false
categories: oracle
tags: 12c
keywords: RMAN  
description: new features in 12c
---
With the multitenant architecture, Oracle 12c provide some great new features. Such as RMAN table-level recovery, moving datafiles online and so on. This post will introduce some of the new features in 12c.
<!--more-->
### 1. RMAN enhancement
Most of previous RMAN commands are still supported in new 12c. In CDB environments, backup the whole CDB it means including backup ALL PDBs. You also can backup individual PDB just like previous version. 
#### 1.1 Table-level recovery
As of 12c, RMAN provide a fast way to recover a dropped/truncated table than before, and this recovery procedure is without any affecting the remaining database objects. Before you can perform the table-level recovery, you must backup the SYSTEM,SYSAUX,UNDO and the relative tablespaces. If the table you want to recover is reside in PDB, you also need to backup the ROOT container's SYSTEM,SYSAUX, UNDO tablespaces along with PDB's SYSTEM,SYSAUX tablespaces.    
The grammar of table-level recovery is like below:
```
RECOVER TABLE schema.tabname OF PLUGGABLE DATABASE pdbname
UNTIL TIME/SCN
AUXILIARY DESTINATION 'auxiliary_dir'
DATAPUMP DESTINATION 'datapump_dir'
#if you want to restore the data into a different table
#please use REMAP TABLE clause, it's suitable for truncated/deleted table
REMAP TABLE schema.t1:t2	
#or import the data manually 
DUMP FILE 'dump_file_name'
NOTABLEIMPORT;
```

- Example of table level-recovery 

Create a table reside in PDB pdb12c.
```
SQL> conn fung@db2srv:1522/pdb12c
SQL> create table user_t as select * from dba_users;
#find the current scn
SQL> select current_scn from v$database;
CURRENT_SCN
-----------
    1868050
SQL> select count(*) from user_t;

  COUNT(*)
----------
        37
```
Perform a CDB full backup first. 
```
[ora12c@db2srv:/home/ora12c]$ cat full.bk.sh 
#!/bin/bash
source ~/.bash_profile 

RMAN=$ORACLE_HOME/bin/rman 
BAKDIR=/u01/backup
LOG=$BAKDIR/fullCDB.`date +%Y%m%d`.log

$RMAN target / <<EOF |tee $LOG
run{
allocate channel t1 type disk format '/u01/backup/full_%d_%T_%s';
backup full database 
include current controlfile
plus archivelog delete all input;
release channel t1;
}
exit
EOF
```
Now, truncate the table 
```
SQL> truncate table user_t;
```
If in production environments, this table maybe still got some DBL operations, in this situation, it's recommended that use remap approach to recover this table. 
```
#connect to CDB with RMAN
RMAN> recover table fung.t of pluggable database pdb12c
until scn 1868050
auxiliary destination '/backup/tmpdb'
remap table fung.user_t:t_recv;
```
Check the result:
```
[ora12c@db2srv:/home/ora12c]$ sqlplus fung/oracle@db2srv:1522/pdb12c
SQL> select count(*) from T_RECV;
  COUNT(*)
----------
	37
```
From the output,  we can find what happened during the recovery:

- Create an auxiliary instance automatically

- Restore the dadafiles into the auxiliary instance 

- Perform a point in time recovery 

- Export the table 

- Import the table to a specified table 

- Do house keeping, delete the instance 

Some restrictions in table-level recovery:

- Additional disk space needed for storing the auxiliary instance 

- SYS user tables cannot be recovered

- REMAP with NOT NULL constraints is not supported 

#### 1.2 Data Recovery Advisor
This feature only support non-DCB or single-database. It provide possible resolutions regarding data loss analyze. 
```
#delete datafile by accidentally 
ASMCMD [+data/ora12c/datafile] > rm -rf FUNG.259.907875155
RMAN> list failure detail;
using target database control file instead of recovery catalog
Database Role: PRIMARY
List of Database Failures
=========================

Failure ID Priority Status    Time Detected       Summary
---------- -------- --------- ------------------- -------
1142       HIGH     OPEN      2016-03-31 16:54:35 One or more non-system datafiles are missing
  Impact: See impact for individual child failures
  List of child failures for parent failure ID 1142
  Failure ID Priority Status    Time Detected       Summary
  ---------- -------- --------- ------------------- -------
  1145       HIGH     OPEN      2016-03-31 16:54:35 Datafile 11: '+DATA/ORA12C/DATAFILE/fung.259.907875155' is missing
    Impact: Some objects in tablespace FUNG might be unavailable
```
Show the advise of the resolutions:
```
RMAN> advise failure 1145;
Database Role: PRIMARY
List of Database Failures

Failure ID Priority Status    Time Detected       Summary
---------- -------- --------- ------------------- -------
1145       HIGH     OPEN      2016-03-31 16:54:35 Datafile 11: '+DATA/ORA12C/DATAFILE/fung.259.907875155' is missing
  Impact: Some objects in tablespace FUNG might be unavailable
analyzing automatic repair options; this may take some time
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=26 device type=DISK
analyzing automatic repair options complete
Mandatory Manual Actions
========================
no manual actions available
Optional Manual Actions
=======================
1. If file +DATA/ORA12C/DATAFILE/fung.259.907875155 was unintentionally renamed or moved, restore it

Automated Repair Options
========================
Option Repair Description
------ ------------------
1      Restore and recover datafile 11  
  Strategy: The repair includes complete media recovery with no data loss
  Repair script: /u01/app/ora12c/diag/rdbms/ora12c/ora12c/hm/reco_467016983.hm
```
The repair script generated by RMAN as below:
```
[ora12c@db2srv ~]$ cat /u01/app/ora12c/diag/rdbms/ora12c/ora12c/hm/reco_467016983.hm
   # restore and recover datafile
   restore ( datafile 11 );
   recover datafile 11;
   sql 'alter database datafile 11 online';
```

Follow the script to repair the failure:
```
RMAN> restore ( datafile 11 );
RMAN> recover datafile 11;
RMAN> sql 'alter database datafile 11 online';
```
If the failure still remain in DRA, you can remove it by manually:
```
RMAN> change failure 1145 closed;
```

#### 1.3 Execute SQLs in RMAN
As of 12c, not like previous version, RMAN can execute SQLs directly without any pre-sql commands, but you cannot change the containers in RMAN session.
```
RMAN> select id,name,phoneno from fung.t1;

        ID NAME          PHONENO
---------- ---------- ----------
     51369 FUNG            13800
RMAN> alter session set container='CDB$ROOT';

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of sql statement command at 03/31/2016 17:24:29
RMAN-06815: cannot change the container in RMAN session.
```
### 2. Invisible column
This feature is like hidden column in DB2 database. Below example is the hidden column in DB2. 
```
db2 "create table t (id int, name char(10), phoneno char(13) implicitly hidden) "
db2 "insert into t (id,name,phoneno) values (51369,'FUNG','13800')"
[db2v97i@db2srv ~]$ db2 "select * from t"

ID          NAME      
----------- ----------
      51369 FUNG  
[db2v97i@db2srv ~]$ db2 "select id,name,phoneno from t"

ID          NAME       PHONENO      
----------- ---------- -------------
      51369 FUNG       13800    
[db2v97i@db2srv ~]$ db2 describe table t
                                Data type                     Column
Column name                     schema    Data type name      Length     Scale Nulls
------------------------------- --------- ------------------- ---------- ----- ------
ID                              SYSIBM    INTEGER                      4     0 Yes   
NAME                            SYSIBM    CHARACTER                   10     0 Yes   
PHONENO                         SYSIBM    CHARACTER                   13     0 Yes  
```
In Oracle 12c, the invisible column just like the DB2's hidden column, the difference is in Oracle 12c, you cannot use describe table to show the invisible columns. 
```
SQL> create table t1(id int,name varchar(10), phoneno number(10) invisible);
SQL> insert into t1(id,name,phoneno) values(51369,'FUNG','13800');
SQL> commit;
SQL> select * from t1;
	ID NAME
---------- --------------------
     51369 FUNG
SQL> select id,name,phoneno from t1;
	ID NAME 		   PHONENO
---------- -------------------- ----------
     51369 FUNG 		     13800
SQL> desc t1;
 Name					   Null?    Type
 ----------------------------------------- -------- ----------------------------
 ID						    NUMBER(38)
 NAME						    VARCHAR2(10)
SQL> select dbms_metadata.get_ddl('TABLE','T1','FUNG') from dual;

DBMS_METADATA.GET_DDL('TABLE','T1','FUNG')
--------------------------------------------------------------------------------

  CREATE TABLE "FUNG"."T1"
   (	"PHONENO" NUMBER(10,0) INVISIBLE, 	#invisible column
	"ID" NUMBER(*,0),
	"NAME" VARCHAR2(10)
   ) SEGMENT CREATION IMMEDIATE
```
External table, temporary table and cluster table are not supported invisible column. 

### 3. Online rename/move datafiles 
Please refer to previous post [Moving Files in Database](/moving-files-in-database.html) .

### 4. DDL logging 
Enable this feature by turning parameter <code>enable_ddl_logging</code> to true. Then the DDL statement can be logged into a XML file. The default DDL log location is <code>$ORACLE_BASE/diag/rdbms/DBNAME/log/ddl</code>
```
SQL> show parameter logg
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
enable_ddl_logging		     boolean		    FALSE
SQL> alter system set enable_ddl_logging=true ;
#Initial a DDL statement
SQL> create index t1 on fung.t1(id);
```
You can find the log under the diag directory:
```
[ora12c@db2srv:/u01/app/ora12c/diag/rdbms/ora12c/ora12c/log/ddl]$ more log.xml 
<msg time='2016-03-31T17:41:26.009+08:00' org_id='oracle' comp_id='rdbms'
 msg_id='kpdbLogDDL:18370:2946163730' type='UNKNOWN' group='diag_adl'
 level='16' host_id='db2srv' host_addr='192.168.56.110'
 version='1'>
 <txt>create index t1 on fung.t1(id)    #DDL logging
 </txt>
</msg>
```
This post only covers some of the 12c new features, more freatures will be discovered in the future's post. 






<b>EOF</b></br>



