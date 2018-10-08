---
layout: post
title: "OGG RMAN Initial Load sample"
date: 2016-06-30 15:54:37
comments: false
categories: oracle
tags: ogg
keywords: Oracle GoldenGate
description: Oracle GoldenGate Initial Load
name: OGG RMAN Initial Load sample
author: Fung Kong
datePublished: June 30, 2016
---

My first experience of deploying OGG was in 2010, we were synchronizing the data from prod server to a report server. Before using OGG, we were using MV table to achieve that. But I didn't maintain OGG since 2013, so I almost forgot everything about the OGG. It's necessary to write down some fundamental steps about how to build an OGG environment.
<!--more-->
## Terminology

- MGR process

    The first process you should start before Extract/Replicat, it's the control process of OGG, need to be ran on both source and target. It performs monitoring, starting other OGG process,managing the trail files and so on.

- Extract process

    This process is performing capturing changed committed data on the source, also can be configured for initialing load.

- Trail files

    Trail files can exist on both source and target. It stores changed data of database continuously to disk files, these files are called trails. Without the Data pump configured, there will be no trail files on the source. The source trail file also known as local trail, the target trail also known as remote trail.
- Data pump process

    The main purpose using pump is protecting Extract process abend or abort from network failure between source and target. This process only reside on the source. Without the pump process, the trail file only exist on the target, from the source end, the changed committed data which Extract process would be stored in the memory, in case of network connectivity broken, the Extract maybe run out of memory and abort.   
    With the pump process, the data that Extract process will be storing into a source trail file, if the network failure, since the captured data moved to the disk, there's no risks for running out of memory, once the connectivity restored, the pump process will read the data from the source trail and send to the target trail.

- Replicat process

    The Replicat process runs on the target system. It read the trail on the target, re-apply the DML or DDL operation to the target system.

## Configuring the OGG on the source system

- Creating GoldenGate users

```sql
SQL> create tablespace goldengate datafile '/u01/linora/goldengate01.dbf' size 100M;
SQL> create user ggsusr identified by oracle default tablespace goldengate account unlock;
SQL> grant resource,connect,dba to ggsusr;
```

- Enabling the archive log mode

```sql
SQL> alter system set LOG_ARCHIVE_DEST_1 = 'LOCATION=/u01/arch' scope=both;
SQL> shutdown immediate
SQL> alter database archivelog;
SQL> alter database open;
SQL> alter system switch logfile;
```

- Enabling the supplemental log

```sql
SQL> select SUPPLEMENTAL_LOG_DATA_MIN,
SUPPLEMENTAL_LOG_DATA_PK,
SUPPLEMENTAL_LOG_DATA_UI,
SUPPLEMENTAL_LOG_DATA_FK,
SUPPLEMENTAL_LOG_DATA_ALL from v$database;

SQL> alter database add supplemental log data ;
SQL> alter database add supplemental log data (primary key, unique,foreign key) columns;
SQL> alter system switch logfile;
```
Ensure supplemental log PK,UI,FK are enabled, the ALL Columns are disabled, if the ALL Columns are enabled, use following sql to close it:
```sql
SQL> alter database drop supplemental log data (ALL) columns;
```

- Enabling the force logging

```sql
SQL> select FORCE_LOGGING from v$database;
SQL> alter database force logging;
```

- Installing the GoldenGate software

Before install the software, we need to add some variables to the profile, add the following line to oracle user's profile:
```bash
export GG_HOME=/gghome
export LD_LIBRARY_PATH=$GG_HOME:$ORACLE_HOME/lib:/lib:/usr/lib
#Unzip the package with oracle user
cd /gghome/
unzip V34339-01.zip
tar -xvf fbo_ggs_Linux_x64_ora11g_64bit.tar
```
Create subdirs for OGG
```bash
[oracle@node1:/gghome]$ ./ggsci
GGSCI (node1) 1> create subdirs
Creating subdirectories under current directory /gghome
Parameter files                /gghome/dirprm: already exists
Report files                   /gghome/dirrpt: created
Checkpoint files               /gghome/dirchk: created
Process status files           /gghome/dirpcs: created
SQL script files               /gghome/dirsql: created
Database definitions files     /gghome/dirdef: created
Extract data files             /gghome/dirdat: created
Temporary files                /gghome/dirtmp: created
Stdout files                   /gghome/dirout: created
```

- Enabling sequence support

```sql
[oracle@node1:/gghome]$ cd /gghome/
[oracle@node1:/gghome]$ sqlplus "/as sysdba"
SQL> @sequence.sql
SQL> GRANT EXECUTE on ggsusr.updateSequence to ggsusr;
SQL> ALTER TABLE sys.seq$ ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
```

- Adding trandata to source tables

```sql
SQL> set head off
SQL> set feedback off
SQL> spool /gghome/dirprm/addtrandata.txt
SQL> select 'add trandata '|| owner ||'.'|| table_name from dba_tables where owner='SCOTT';

add trandata SCOTT.DEPT
add trandata SCOTT.EMP
add trandata SCOTT.BONUS
add trandata SCOTT.SALGRADE
SQL> spool off

[oracle@node1:/home/oracle]$ cd /gghome/
[oracle@node1:/gghome]$ ./ggsci
GGSCI (node1) 1> dblogin userid ggsusr, password oracle
Successfully logged into database.
GGSCI (node1) 3> obey /gghome/dirprm/addtrandata.txt
#find the result of adding trandata
GGSCI (node1) 11> info trandata scott.dept
```

- Configuring manager process

```bash
#Encrept the passowrd
GGSCI (node1) 2> ENCRYPT PASSWORD oracle BLOWFISH ENCRYPTKEY DEFAULT
Using default key...

Encrypted password:  AACAAAAAAAAAAAGAIFAAUDVHCFUGFIYF
Algorithm used:  BLOWFISH

GGSCI (node1) 3> edit params mgr
PORT 7809
DYNAMICPORTLIST  7810-7880
USERID ggsusr, PASSWORD AAACAAAAAAAAAAAGAIFAAUDVHCFUGFIYF, ENCRYPTKEY default
AUTORESTART EXTRACT *, RETRIES 5, WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPDAYS 3
--PURGEOLDEXTRACTS parameter is determinating how long the extract trail file is keeping
--Disable this parameter during initail load, and enable it when sychronizate
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```

- Configuring Extract process

```bash
#We need to confirm we don't have any transactions running, otherwise, rollback it or waiting for completing.
SQL> select count(*) from gv$transaction;
GGSCI (node1) 5> dblogin userid ggsusr, password oracle
Successfully logged into database.
GGSCI (node1) 6> edit params ext1
GGSCI (node1) 7> view params ext1

EXTRACT ext1
--SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
USERID ggsusr, PASSWORD AAACAAAAAAAAAAAGAIFAAUDVHCFUGFIYF, ENCRYPTKEY default
GETTRUNCATES
REPORTCOUNT EVERY 1 MINUTES, RATE
DISCARDFILE ./dirrpt/ext1.dsc, APPEND, MEGABYTES 1024
WARNLONGTRANS 2h, CHECKINTERVAL 3m
EXTTRAIL ./dirdat/la
DYNAMICRESOLUTION
DBOPTIONS  ALLOWUNUSEDCOLUMN
FETCHOPTIONS NOUSESNAPSHOT
--TRANLOGOPTIONS altarchivelogdest primary instance RACDB1 /RAC1_arch1, altarchivelogdest  instance  RACDB2 /RAC2_arch1
--THREADOPTIONS   MAXCOMMITPROPAGATIONDELAY 60000 IOLATENCY 60000
--TRANLOGOPTIONS is for RAC, tell OGG where the location to extract data from archive log desc
--table----
obey ./dirprm/source_tables.txt
```

Generate tables using following SQLs:
```sql
SQL> set termout off
set feedback off
set verify off
set pagesize 0
spool /gghome/dirprm/source_tables.txt
SQL> select 'table '||owner||'.'||table_name||';' from dba_tables where owner='SCOTT';
table SCOTT.DEPT;
table SCOTT.EMP;
table SCOTT.BONUS;
table SCOTT.SALGRADE;
SQL> spool off
```

- Configuring Data pump process

```sql
GGSCI (node1) 12> edit params dpe1
GGSCI (node1) 13> view params dpe1
EXTRACT dpe1
PASSTHRU
RMTHOST 192.168.56.102, MGRPORT 7809, compress
DYNAMICRESOLUTION
RMTTRAIL ./dirdat/ra
--table----
obey ./dirprm/source_tables.txt
```

- Adding Extract and Data pump trail file

```sql
GGSCI (node1) 14> add extract ext1, tranlog, begin now
EXTRACT added.
--RAC use: add extract ext1, tranlog, begin now threads 2
--threads means how much nodes of this RAC
GGSCI (node1) 16> add exttrail ./dirdat/la, extract ext1, megabytes 200
EXTTRAIL added.
GGSCI (node1) 18> add extract dpe1, exttrailsource ./dirdat/la
EXTRACT added.
GGSCI (node1) 19> add rmttrail ./dirdat/ra, extract dpe1, megabytes 200
RMTTRAIL added.
GGSCI (node1) 20> start mgr
GGSCI (node1) 21> start ext1
```


## Performing full online backup of the source database

- Pre-backup actions

```sql
SQL> select count(*) from gv$transaction;
#If no transactions running, do the full backup
SQL> select dbid from v$database;
      DBID
      ----------
      3461558107
SQL> select 'set newname for datafile  '||file_id||' to '''||'/u01/linora/temp01.dbf'||''';' from dba_temp_files;
set newname for datafile  1 to '/u01/linora/temp01.dbf';

SQL> select 'set newname for datafile '||file# ||' to '''||'/u01/linora'||''';' from v$datafile;
set newname for datafile 1 to '/u01/linora/system01.dbf';
set newname for datafile 2 to '/u01/linora/sysaux01.dbf';
set newname for datafile 3 to '/u01/linora/undotbs01.dbf';
set newname for datafile 4 to '/u01/linora/users01.dbf';
set newname for datafile 5 to '/u01/linora/goldengate01.dbf';

#Copy the password from source to target
scp $ORACLE_HOME/dbs/orapwlinora node2:$ORACLE_HOME/dbs/

#Record source redo log location
SQL> SELECT member FROM gv$logfile;

MEMBER
--------------------------------------------------------------------------------
/u01/linora/redo03.log
/u01/linora/redo02.log
/u01/linora/redo01.log
```

- Performing full backup

```bash
#!/bin/bash
. ~/.bash_profile
export DB_NAME=linora
RMAN=$ORACLE_HOME/bin/rman
SQLPLUS=$ORACLE_HOME/bin/sqlplus
TEE=/usr/bin/tee
DBDEST=/backup
LOGFILE=/backup/log/backup_db_`date +'%Y%m%d%H%M%S'`.log
mkdir -p ${DBDEST}/log
echo "--------------------backup start `date +'%Y-%m-%d %H:%M:%S'`--------------------" > $LOGFILE
$RMAN <<EOF | $TEE -a $LOGFILE
connect target
run {
ALLOCATE CHANNEL ch00 TYPE DISK MAXPIECESIZE 200G;

ALLOCATE CHANNEL ch01 TYPE DISK MAXPIECESIZE 200G;
CROSSCHECK BACKUPSET;
DELETE NOPROMPT EXPIRED BACKUPSET;
sql 'alter system archive log current';
BACKUP AS BACKUPSET SKIP INACCESSIBLE
TAG hot_db_bk_level0 FORMAT '/backup/bk_%s_%p_%U_%T_%d'
FULL  DATABASE;
backup current controlfile tag 'ctl' format '/backup/ctl_%U_%T_%d';
RELEASE CHANNEL ch00;
RELEASE CHANNEL ch01;
}
exit;
EOF
echo "--------------------backup end `date +'%Y-%m-%d %H:%M:%S'`--------------------" >> $LOGFILE
exit 0
```

- Performing redo and archive log backup plus control file

```bash
#!/bin/bash
. ~/.bash_profile
export DB_NAME=linora
RMAN=$ORACLE_HOME/bin/rman
SQLPLUS=$ORACLE_HOME/bin/sqlplus
TEE=/usr/bin/tee
DBDEST=/backup
LOGFILE=/backup/log/backup_archivelog_`date +'%Y%m%d%H%M%S'`.log
mkdir -p ${DBDEST}

echo "--------------------backup start `date +'%Y-%m-%d %H:%M:%S'`--------------------" > $LOGFILE
$RMAN <<EOF | $TEE -a $LOGFILE
connect target
run {
ALLOCATE CHANNEL ch00 TYPE DISK MAXPIECESIZE 200G;
ALLOCATE CHANNEL ch01 TYPE DISK MAXPIECESIZE 200G;
sql 'alter system switch logfile';
sql 'alter system switch logfile';
sql 'alter system switch logfile';
sql 'alter system archive log current';
BACKUP ARCHIVELOG FROM TIME 'sysdate-3' FORMAT '/backup/ARCH_%U_T_%d';
RELEASE CHANNEL ch00;
RELEASE CHANNEL ch01;
}
exit;
EOF
echo "--------------------backup end `date +'%Y-%m-%d %H:%M:%S'`--------------------" >> $LOGFILE

exit 0
```

- Retrieving the SCN after backing up the database and scp the backup images to node2

```sql
SQL> select inst_id,group#,thread#,sequence#,members,status,first_change#,next_change# from gv$log;

INST_ID     GROUP#    THREAD#  SEQUENCE#    MEMBERS STATUS                           FIRST_CHANGE# NEXT_CHANGE#
---------- ---------- ---------- ---------- ---------- -------------------------------- ------------- ------------
1          1          1         13          1 INACTIVE                               1007503      1007512
1          2          1         14          1 INACTIVE                               1007512      1007521
1          3          1         15          1 CURRENT                                1007521   2.8147E+14
```

We use the largest SCN of the inactive member. Here's 1007512.

## Restoring the database to target system
- Modifying the initial parameter file

```bash
mkdir -p /u01/app/oracle/admin/linora/adump
mkdir -p /u01/linora/
mkdir -p /u01/app/oracle/fast_recovery_area/linora/
mkdir -p /u01/arch
mkdir -p /u01/app/oracle
```

- Starting up the instance to nomount mode

```sql
SQL> startup nomount
```

- Restoring the control file and mount the database

```bash
RMAN> restore controlfile from '/backup/ctl_0cr9c7na_1_1_20160629_LINORA';
Starting restore at 2016-06-29 16:02:25
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=18 device type=DISK
channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
output file name=/u01/linora/control01.ctl
output file name=/u01/app/oracle/fast_recovery_area/linora/control02.ctl
Finished restore at 2016-06-29 16:02:27

RMAN> sql 'alter database mount';
sql statement: alter database mount
released channel: ORA_DISK_1
```

- Cataloging the backup images

```sql
RMAN> catalog start with '/backup';
```

- Restoring and recovering the database

```sql
#!/bin/bash
. ~/.bash_profile
export DB_NAME=linora
RMAN=$ORACLE_HOME/bin/rman
SQLPLUS=$ORACLE_HOME/bin/sqlplus
TEE=/usr/bin/tee
DBDEST=/backup
LOGFILE=/backup/log/restore_db_`date +'%Y%m%d%H%M%S'`.log
mkdir -p ${DBDEST}/log

echo "--------------------restore start `date +'%Y-%m-%d %H:%M:%S'`--------------------" > $LOGFILE
$RMAN <<EOF | $TEE -a $LOGFILE
connect target
run{
allocate channel c0 type disk;
allocate channel c1 type disk;
set newname for datafile 1 to '/u01/linora/system01.dbf';
set newname for datafile 2 to '/u01/linora/sysaux01.dbf';
set newname for datafile 3 to '/u01/linora/undotbs01.dbf';
set newname for datafile 4 to '/u01/linora/users01.dbf';
set newname for datafile 5 to '/u01/linora/goldengate01.dbf';
set newname for tempfile 1 to '/u01/linora/temp01.dbf';
restore database; 
switch datafile all;
switch tempfile all;
release channel c0;
release channel c1;
}
exit;
EOF
echo "--------------------restore end `date +'%Y-%m-%d %H:%M:%S'`--------------------" >> $LOGFILE

exit 0
```

If needed, rename the redo log file:
```sql
SQL> SELECT member FROM gv$logfile;
SQL> alter database rename file '/u01/.../redo01.log' to '/u02/.../redo01.log';
```

Recover the database from specify SCN  1007512.
```sql
RMAN> recover database until scn 1007512;
```

Check the file header and the control file SCN
```sql
SQL> select checkpoint_change# from v$datafile;
CHECKPOINT_CHANGE#
------------------
1007512
1007512
1007512
1007512
1007512

SQL> select checkpoint_change# from v$datafile_header;
CHECKPOINT_CHANGE#
------------------
1007512
1007512
1007512
1007512
1007512
```

If we don't have temp tablespace, we need to create the TBS:
```sql
SQL> select name,bytes/1024/1024/1024 GB from v$tempfile;
SQL> create temporary tablespace TEMP tempfile '/u01/linora/temp01.dbf' size 32G;
SQL> alter database default temporary tablespace TEMP;
SQL> alter database open resetlogs;
--using spfile instead pfile
SQL> create spfile from pfile;
SQL> shutdown immediate
SQL> startup
```

## Configuring the OGG on the target system

- Creating the Golden Gate user if we don't have one

```sql
SQL> create tablespace goldengate datafile '+DATA' size 5000M autoextend on next 64M maxsize unlimited;
SQL> create user goldengate identified by goldengate default tablespace goldengate temporary tablespace TEMP profile DEFAULT;
SQL> grant dba to goldengate;
```

- Disabling the trigger on the target database

Disable trigger SP:

```sql
declare
v_sql varchar2(2000);
CURSOR c_trigger IS SELECT 'alter trigger '||owner||'.'||trigger_name||' disable' from dba_triggers where status='ENABLED' and owner in ('SCOTT');
BEGIN
OPEN c_trigger;
LOOP
FETCH c_trigger INTO v_sql;
EXIT WHEN c_trigger%NOTFOUND;
execute immediate v_sql;
end loop;
close c_trigger;
end;
/
```

Enable trigger SP:

```sql
declare
v_sql varchar2(2000);
CURSOR c_trigger IS SELECT 'alter trigger '||owner||'.'||trigger_name||' enable' from dba_triggers where status='ENABLED' and owner in ('SCOTT');
BEGIN
OPEN c_trigger;
LOOP
FETCH c_trigger INTO v_sql;
EXIT WHEN c_trigger%NOTFOUND;
execute immediate v_sql;
end loop;
close c_trigger;
end;
/
```

- Disabling the foreign key

Disable FK SP:

```sql
declare
v_sql varchar2(2000);
CURSOR c_trigger IS SELECT 'alter table '||owner||'.'||table_name||' disable constraint '||constraint_name from dba_constraints where constraint_type='R' and owner in ('SCOTT');
BEGIN
OPEN c_trigger;
LOOP
FETCH c_trigger INTO v_sql;
EXIT WHEN c_trigger%NOTFOUND;
execute immediate v_sql;
end loop;
close c_trigger;
end;
/
```

Enable FK SP:

```sql
declare
v_sql varchar2(2000);
CURSOR c_trigger IS SELECT 'alter table '||owner||'.'||table_name||' enable constraint '||constraint_name from dba_constraints where constraint_type='R' and owner in ('SCOTT');
BEGIN
OPEN c_trigger;
LOOP
FETCH c_trigger INTO v_sql;
EXIT WHEN c_trigger%NOTFOUND;
execute immediate v_sql;
end loop;
close c_trigger;
end;
/
```

- Disabling the jobs

When target start the replication, broken the DML job, but keep the MV job running.

```sql
SQL> set line 200
SQL> col what for a50
SQL> select job,log_user what from dba_jobs;
SQL> select log_user,job,what from dba_jobs where what like '%refresh%';
conn /as sysdba
SQL>exec dbms_ijob.broken(121,true);
SQL>exec dbms_ijob.broken(141,true);
SQL>exec dbms_ijob.broken(301,true);
```

- Configuring the listener on the target system

```sql
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = PLSExtProc)
      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/db_1)
      (PROGRAM = extproc)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = linora)
      (ORACLE_HOME = u01/app/oracle/product/11.2.0/db_1)
      (SID_NAME = linora)
    )
  )

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = node2)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

ADR_BASE_LISTENER = /u01/app/oracle
```

- Other operations before installing the OGG

Disable archival on target system
```sql
SQL> shutdown immediate
SQL> startup mount
SQL> alter database noarchivelog;
SQL> alter database open;
```

If we have sequence support on OGG, we need to grant necessary privilege to target database.

```sql
GRANT EXECUTE on ggsusr.replicateSequence TO ggsusr;
exec dbms_streams_auth.grant_admin_privilege('ggsusr')
```

Disable sys user capture DDL
```sql
Alter trigger sys.GGS_DDL_TRIGGER_BEFORE disable;
```

- Installing OGG
Unzip and tar the OGG package. The same operation as the source.

- Configuring GLOBAL parameters

```sql
[oracle@node2:/u01/arch]$ cd /gghome/
[oracle@node2:/gghome]$ ./ggsci 
GGSCI (node2) 2> create subdirs
GGSCI> EDIT PARAMS ./GLOBALS
CHECKPOINTTABLE ggsusr.checktable
```

- Enabling sequence support

```sql
[oracle@node2:/backup]$ cd /gghome/
[oracle@node2:/gghome]$ sqlplus "/as sysdba"
SQL> @sequence.sql
SQL> grant execute on ggsusr.replicateSequence to ggsusr;
```

- Configuring manager process

```sql
GGSCI (node2) 4> edit params mgr
PORT 7809
DYNAMICPORTLIST 7810-7880
USERID ggsusr, PASSWORD AACAAAAAAAAAAAGAIFAAUDVHCFUGFIYF, ENCRYPTKEY default
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints,minkeepdays 3
--PURGEOLDEXTRACTS parameter is determinating how long the extract trail file is keeping
--Disable this parameter during initail load, and enable it when sychronizate
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```

- Configuring Replicat process

```sql
GGSCI (node2) 5> dblogin userid ggsusr, password oracle
GGSCI (node2) 6> add checkpointtable ggsusr.checktable
GGSCI (node2) 7> ENCRYPT PASSWORD oracle BLOWFISH ENCRYPTKEY DEFAULT
Using default key...
Encrypted password:  AACAAAAAAAAAAAGAIFAAUDVHCFUGFIYF

GGSCI (node2) 8> edit params repy
REPLICAT repy
--setenv (NLS_LANG=AMERICAN_AMERICA.ZHS16GBK)
USERID ggsusr, PASSWORD AACAAAAAAAAAAAGAIFAAUDVHCFUGFIYF, ENCRYPTKEY default
REPORT AT 01:59
REPORTCOUNT EVERY 30 MINUTES, RATE
REPERROR DEFAULT, ABEND
DBOPTIONS DEFERREFCONST
assumetargetdefs
DISCARDFILE ./dirrpt/repy.dsc, APPEND, MEGABYTES 1024
DISCARDROLLOVER AT 02:30
GETTRUNCATES 
ALLOWNOOPUPDATES
--table--
obey ./dirprm/map_tables_repy.txt
```

You can find the map_tables_repy.txt from below SQLs:
```sql
SQL> select 'MAP '||owner||'.'||table_name||', TARGET '||owner||'.'||table_name||';' from dba_tables where owner='SCOTT';
'MAP'||OWNER||'.'||TABLE_NAME||',TARGET'||OWNER||'.'||TABLE_NAME||';'
----------------------------------------------------------------------------
MAP SCOTT.DEPT, TARGET SCOTT.DEPT;
MAP SCOTT.EMP, TARGET SCOTT.EMP;
MAP SCOTT.SALGRADE, TARGET SCOTT.SALGRADE;
MAP SCOTT.BONUS, TARGET SCOTT.BONUS;
```

Add replicat process:
```sql
GGSCI (node2) 10> ADD REPLICAT repy, EXTTRAIL ./dirdat/ra, checkpointtable ggsusr.checktable
REPLICAT added.
```

- Starting the replication

Start manager process on the target:
```sql
GGSCI (node2) 1> start mgr
```

Start the pump process on the source, and observe if any trail files are delivery to the target:
```sql
GGSCI (node1) 4> start dpe1
```

Start the replicat process on the target:
```sql
GGSCI (node2) 7> start repy aftercsn 1007521;
```

## Testing the replication
Update one of the recode of scott.emp table
```sql
SQL> update scott.emp set SAL=1000 where empno = 7900;
1 row updated.
SQL> commit;
Commit complete.
SQL> select * from scott.emp where empno=7900;
EMPNO ENAME                JOB                       MGR HIREDATE                   SAL       COMM     DEPTNO
---------- -------------------- ------------------ ---------- ------------------- ---------- ---------- ----------
7900 JAMES                CLERK                    7698 1981-12-03 00:00:00       1000                    30
```

After a few minutes, check the result from the target.
```sql
SQL> select * from scott.emp where empno=7900;
EMPNO ENAME                JOB                       MGR HIREDATE                   SAL       COMM     DEPTNO
---------- -------------------- ------------------ ---------- ------------------- ---------- ---------- ----------
7900 JAMES                CLERK                    7698 1981-12-03 00:00:00       1000                    30
```

## Enable DDL replication
Before enable ddl replication, we must ensure:

- Recyclebin has been disabled(10g).
- The target and source schema must be identical.

Enable DDL support
```sql
[oracle@node1:/home/oracle]$ cd /gghome/
[oracle@node1:/gghome]$ sqlplus "/as sysdba"
SQL> grant execute on utl_file to ggsusr;
SQL> @marker_setup.sql
SQL> @ddl_setup.sql
SQL> @role_setup.sql
SQL> GRANT GGS_GGSUSER_ROLE TO ggsusr;
SQL> @ddl_enable.sql
```

Edit the OGG params:
```sql
--Both on source and target
GGSCI (node1) 12> edit params GLOBALS
GGSCHEMA ggsusr

--add the folloing line to Extract and Replicat.
GGSCI (node1) 14> edit param ext1
ddl include mapped
ddloptions addtrandata, report

--Only on target
GGSCI (node2) 8> edit params repy
DDLERROR DEFAULT IGNORE
```

Now, have a test:
```sql
--On the source, modify the column
SQL> desc scott.emp
Name                                                              Null?    Type
----------------------------------------------------------------- -------- ---------------------
EMPNO                                                             NOT NULL NUMBER(4)
ENAME                                                                      VARCHAR2(10)
JOB                                                                        VARCHAR2(9)
MGR                                                                        NUMBER(4)
HIREDATE                                                                   DATE
SAL                                                                        NUMBER(7,2)
COMM                                                                       NUMBER(7,2)
DEPTNO                                                                     NUMBER(2)
SQL> alter table scott.emp add ncol varchar2(10);
SQL> create table scott.test as select * from scott.emp;

--On target
SQL> select count(*) from scott.test;
  COUNT(*)
  ----------
          14
```

## Trouble shooting

- When doing some DMLs to the table, encountered following errors:

```bash
2016-06-29 21:43:43  WARNING OGG-00869  Oracle GoldenGate Delivery for Oracle, repy.prm:  OCI Error ORA-26945: unsupported hint RESTRICT_ALL_REF_CONS (status = 26945). UPDATE /*+ RESTRICT_ALL_REF_CONS */ "SCOTT"."EMP" SET "SAL" = :a1 WHERE "EMPNO" = :b0.
2016-06-29 21:43:43  WARNING OGG-01004  Oracle GoldenGate Delivery for Oracle, repy.prm:  Aborted grouped transaction on 'SCOTT.EMP', Database error 26945 (OCI Error ORA-26945: unsupported hint RESTRICT_ALL_REF_CONS (status = 26945). UPDATE /*+ RESTRICT_ALL_REF_CONS */ "SCOTT"."EMP" SET "SAL" = :a1 WHERE "EMPNO" = :b0).
2016-06-29 21:43:43  WARNING OGG-01003  Oracle GoldenGate Delivery for Oracle, repy.prm:  Repositioning to rba 1479 in seqno 0.
2016-06-29 21:43:43  WARNING OGG-01154  Oracle GoldenGate Delivery for Oracle, repy.prm:  SQL error 26945 mapping SCOTT.EMP to SCOTT.EMP OCI Error ORA-26945: unsupported hint RESTRICT_ALL_REF_CONS (status = 26945). UPDATE /*+ RESTRICT_ALL_REF_CONS */ "SCOTT"."EMP" SET "SAL" = :a1 WHERE "EMPNO" = :b0.
```

Solution:
```sql
SQL> exec dbms_goldengate_auth.grant_admin_privilege('GGSUSR');
SQL> ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION = TRUE SCOPE = BOTH;
```

If you want to skip current transaction, do the following:
```sql
GGSCI (node2) 33> START REPLICAT repy SKIPTRANSACTION
```

- When enabling DDL support, encountered following errors:

```sql
SQL> @ddl_setup.sql
[oracle@node1:/gghome]$ more ddl_setup_spool.txt
...
BEGIN "GGSUSR" .initial_setup; END;
*
*ERROR at line 1:
ORA-20782: Creating GGS_DDL_RULES table:ORA-01031: insufficient privileges:ORA-01031: insufficient privileges
ORA-06512: at "GGSUSR.INITIAL_SETUP", line 477
ORA-06512: at line 1
```

Solution:
```sql
cd /gghome/
sqlplus "/as sysdba"
SQL> @ddl_disable.sql
SQL> grant create table,create sequence to ggsusr;
SQL> @ddl_setup.sql
```

</br><b>EOF</b></br>
