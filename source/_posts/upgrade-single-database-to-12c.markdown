---
layout: post
title: "Upgrade single database to 12c"
date: 2016-03-25 15:03:33
comments: false
categories: oracle
tags: upgrade, 12c
keywords: upgrade Oracle to 12c
description: upgrade Oracle database to 12c and convert it to PDB
---
There are two upgrade approaches, one is with GUI tool DBUA, another is upgrade with manually. This post is focus on manualy upgrade method. After upgrade to 12c, the new version database is non-CDB,which is DEPRECATED by Oracle, so I also will show you how to convert the non-CDB to PDB.
<!--more-->
Regarding the PDB and CDB, you can refer my previous topic [CDB Overview](/cdb-overview.html).
DBUA can automatic backup the database, when upgrade finished, if you're not satisfied with the result, you can simply restore to previous version if you backup the database with DBUA.
### 1. Pre-upgrade tasks
Database version 11.2.0.2 or higher, 10.2.0.5 can be upgraded to 12c directly. Other version should be upgraded to the specific version to meet upgrade  requirements.
Assume my new 12c binary code has been install into a new directory: /u02/app/oracle/product/12.1.0/db1/.
#### 1.1 Perform an offline full backup before upgrade
Upgrade is high risk operations in any softwares, whenever you upgrade a database, you should perform a full backup.
```
SQL> shutdown immediate
SQL> startup mount
SQL> create pfile from spfile;
RMAN> run {
allocate channel c1 type disk format '/backup/full_offline_%d_%T_%s';
backup full database
include current controlfile;
release channel c1;
}
```
#### 1.2 Pre-request check

- Gather Statistics

This step can reduce the amount of downtime.
```
SQL> alter database open;
SQL> EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
```

- Purge recyclebin

```
SQL> purge recyclebin;
```

- Check the integrity of the source database

```
SQL> @?/rdbms/admin/utlrp.sql
```

#### 1.3 Run the preupgrade.sql from 12c
These scripts are copied from from 12c <code>$ORACLE_HOME/rdbms/admin</code>.
```
[oracle@linora:/home/oracle]$ cp /u02/app/oracle/product/12.1.0/db1/rdbms/admin/preupgrd.sql ~/
[oracle@linora:/home/oracle]$ cp /u02/app/oracle/product/12.1.0/db1/rdbms/admin/utluppkg.sql ~/
SQL> @preupgrd.sql
--sample of output
ACTIONS REQUIRED:
1. Review results of the pre-upgrade checks:
 /u01/app/oracle/cfgtoollogs/linora/preupgrade/preupgrade.log
2. Execute in the SOURCE environment BEFORE upgrade:
 /u01/app/oracle/cfgtoollogs/linora/preupgrade/preupgrade_fixups.sql
3. Execute in the NEW environment AFTER upgrade:
 /u01/app/oracle/cfgtoollogs/linora/preupgrade/postupgrade_fixups.sql
```
The Pre-Upgrade Information Tool produces 3 scripts.

- preupgrade.log: The results of all the checks performed. You need to check this to see if it is safe to continue with the upgrade.

- preupgrade_fixups.sql: A fixup script that should be run before the upgrade.

- postupgrade_fixups.sql: A fixup script that should be run after the upgrade.

So, let's run preupgrade fix script.
```
SQL> @/u01/app/oracle/cfgtoollogs/linora/preupgrade/preupgrade_fixups.sql
```
#### 1.4 Copy source database configuration files to new 12c ORACLE_HOME
Copy the netowrk configuration files, the password files into the new DBHOME. Modify network config files to meet the new database configuration.
```
[oracle@linora:/home/oracle]$ export ORACLE_12C_HOME=/u02/app/oracle/product/12.1.0/db1
[oracle@linora:/home/oracle]$ cp $ORACLE_HOME/network/admin/*.ora $ORACLE_12C_HOME/network/admin/
[oracle@linora:/home/oracle]$ cp $ORACLE_HOME/dbs/orapw* $ORACLE_12C_HOME/dbs
[oracle@linora:/home/oracle]$ cp $ORACLE_HOME/dbs/*.ora $ORACLE_12C_HOME/dbs
```
After that, stop the listener, stop the dbconsole and isqlplus.
```
LSNRCTL> stop
[oracle@linora:/home/oracle]$ emctl stop dbconsole
[oracle@linora:/home/oracle]$ isqlplusctl stop
#Remove em when you are using old version EM
[oracle@linora:/home/oracle]$ source ~/.bash_profile_11g
[oracle@linora:/home/oracle]$ sqlplus "/as sysdba"
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
SQL> @/u02/app/oracle/product/12.1.0/db1/rdbms/admin/emremove.sql
old  69:     IF (upper('&LOGGING') = 'VERBOSE')
new  69:     IF (upper('VERBOSE') = 'VERBOSE')
PL/SQL procedure successfully completed.
```
Remove OLAP if you are using in old version, because OALP catalog is desupported in 12c.
```
[oracle@linora:/home/oracle]$ source ~/.bash_profile_11g
[oracle@linora:/home/oracle]$ sqlplus "/as sysdba"
SQL> @?/olap/admin/catnoamd.sql
```
Before upgrade, check the dba_registry:
```
SQL> set line 200
col comp_name for a50
col version for a15
col status for a10
select comp_name,version,status from dba_registry;
COMP_NAME                                          VERSION         STATUS
-------------------------------------------------- --------------- ----------
OWB                                                11.2.0.4.0      VALID
Oracle Application Express                         3.2.1.00.12     VALID
Spatial                                            11.2.0.4.0      VALID
Oracle Multimedia                                  11.2.0.4.0      VALID
Oracle XML Database                                11.2.0.4.0      VALID
Oracle Text                                        11.2.0.4.0      VALID
Oracle Expression Filter                           11.2.0.4.0      VALID
Oracle Rules Manager                               11.2.0.4.0      VALID
Oracle Workspace Manager                           11.2.0.4.0      VALID
Oracle Database Catalog Views                      11.2.0.4.0      VALID
Oracle Database Packages and Types                 11.2.0.4.0      VALID
JServer JAVA Virtual Machine                       11.2.0.4.0      VALID
Oracle XDK                                         11.2.0.4.0      VALID
Oracle Database Java Packages                      11.2.0.4.0      VALID
OLAP Analytic Workspace                            11.2.0.4.0      VALID
Oracle OLAP API                                    11.2.0.4.0      VALID
```

Before upgrade, it's recommended that change the archive log mode to no-archivelog mode.
```
SQL> shutdown immediate
SQL> startup mount
SQL> alter database noarchivelog;
SQL> archive log list;
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            /arch/linora
SQL> shutdown immediate
```
### 2. Upgrade tasks
Update the /etc/oratab to new oracle home, and disable the automatic startup.
```
[oracle@linora:/home/oracle]$ tail -2 /etc/oratab
#linora:/u01/app/oracle/product/11gr2:N
linora:/u02/app/oracle/product/12.1.0/db1:N
```
After modified the oratab file, execute the oraenv to set new environments.
```
[oracle@linora:/home/oracle]$ . oraenv
ORACLE_SID = [linora] ?
The Oracle base remains unchanged with value /u02/app/oracle
```
Now, we can start upgrade now. Although I  keep a copy of old version .bash_profile, but during upgrade,  you need to use new 12c profile.
Execute upgrade.
```
[oracle@linora:/home/oracle]$ source ~/.bash_profile_12c
SQL> startup upgrade
ORACLE instance started.

Total System Global Area  591396864 bytes
Fixed Size		    2927096 bytes
Variable Size		  180356616 bytes
Database Buffers	  402653184 bytes
Redo Buffers		    5459968 bytes
Database mounted.
Database opened.
SQL> exit
```
Execute Upgrade Utilities.
```
[oracle@linora:/home/oracle]$ echo $ORACLE_HOME
/u02/app/oracle/product/12.1.0/db1
[oracle@linora:/home/oracle]$ cd $ORACLE_HOME/rdbms/admin
#the parameter -n, it's value means the parallelism is 4
[oracle@linora:/u02/app/oracle/product/12.1.0/db1/rdbms/admin]$ $ORACLE_HOME/perl/bin/perl catctl.pl -n 4 catupgrd.sql

#portion output
Analyzing file catupgrd.sql
Log files in /u02/app/oracle/product/12.1.0/db1/rdbms/admin
catcon: ALL catcon-related output will be written to catupgrd_catcon_2980.lst
catcon: See catupgrd*.log files for output generated by scripts
catcon: See catupgrd_*.lst files for spool files, if any
...
LOG FILES: (catupgrd*.log)
Upgrade Summary Report Located in:
/u02/app/oracle/product/12.1.0/db1/cfgtoollogs/linora/upgrade/upg_summary.log
Grand Total Upgrade Time:    [0d:0h:15m:44s]
```
### 3. Post-upgrade tasks
Run the post-upgrade scripts when upgrade finished.
```
SQL> startup
#Run catcon.pl to invoke utlrp.sql to recompile any remaining stored PL/SQL and Java code.
[oracle@linora:/home/oracle]$ cd $ORACLE_HOME/rdbms/admin
[oracle@linora:/u02/app/oracle/product/12.1.0/db1/rdbms/admin]$ $ORACLE_HOME/perl/bin/perl \
catcon.pl -n 1 -e -b utlrp -d '''.''' utlrp.sql

catcon: ALL catcon-related output will be written to utlrp_catcon_5779.lst
catcon: See utlrp*.log files for output generated by scripts
catcon: See utlrp_*.lst files for spool files, if any
```
Run postupgrade_fixups.sql.
```
SQL> startup
SQL> @/u01/app/oracle/cfgtoollogs/linora/preupgrade/postupgrade_fixups.sql
```
Run utlu121s.sql to verify that all issues have been fixed.
```
SQL> @?/rdbms/admin/utlu121s.sql
```
Run catuppst.sql
```
SQL> @?/rdbms/admin/catuppst.sql
```
Gether statistics
```
SQL> EXECUTE DBMS_STATS.gather_fixed_objects_stats;
```
Verify and compile invalid objects
```
SQL> @?/rdbms/admin/utlrp.sql
SQL> @?/rdbms/admin/utluiobj.sql
SQL> @?/rdbms/admin/utlu121s.sql
```
Bring the database back to archive log mode.
```
SQL> shutdown immediate
SQL> startup mount
SQL> alter database archivelog;
SQL> alter database open;
SQL> archive log list;
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       /arch/linora
Oldest online log sequence     54
Next log sequence to archive   56
Current log sequence	       56
```
Verify the upgrade result.
```
SQL> set line 200
SQL> col comp_name for a50
SQL> col version for a15
SQL> col status for a10
SQL> select comp_name,version,status from dba_registry;
COMP_NAME                                          VERSION         STATUS
-------------------------------------------------- --------------- ----------
Oracle Application Express                         4.2.5.00.08     VALID
OWB                                                11.2.0.4.0      VALID
Spatial                                            12.1.0.2.0      VALID
Oracle Multimedia                                  12.1.0.2.0      VALID
Oracle XML Database                                12.1.0.2.0      VALID
Oracle Text                                        12.1.0.2.0      VALID
Oracle Workspace Manager                           12.1.0.2.0      VALID
Oracle Database Catalog Views                      12.1.0.2.0      VALID
Oracle Database Packages and Types                 12.1.0.2.0      VALID
JServer JAVA Virtual Machine                       12.1.0.2.0      VALID
Oracle XDK                                         12.1.0.2.0      VALID
Oracle Database Java Packages                      12.1.0.2.0      VALID
OLAP Analytic Workspace                            12.1.0.2.0      VALID
Oracle OLAP API                                    12.1.0.2.0      VALID


```
Backup the database and do rest tasks for post upgrading.
```
RMAN> run{
allocate channel c1 type disk format '/backup/full_%d_%T_%s';
backup full database
include current controlfile
plus archivelog  delete all input;
release channel c1;
}

```
Modify initial file, adjust parameters to meet new 12 database.
```
SQL> shutdown immediate
#adjust oracle_base, compatible, diagnostic_dest ,audit_file_dest etc.,
#do not forget create necessary directories
[oracle@linora:/home/oracle]$ mkdir -p /u02/app/oracle/admin/linora/adump
SQL> startup
SQL> create spfile from pfile;
```
Re-create password file by using orapwd tools.
```
[oracle@linora:/home/oracle]$ orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID \
password=oracle entries=5 force=y ignorecase=y
```
#### 3.1 Upgrade the Time Zone File Version.
```
#current timezone file version
SQL> SELECT version FROM v$timezone_file;
   VERSION
----------
	14
SQL> SELECT PROPERTY_NAME, SUBSTR(property_value, 1, 30) value
FROM DATABASE_PROPERTIES
WHERE PROPERTY_NAME LIKE 'DST_%'
ORDER BY PROPERTY_NAME;

PROPERTY_NAME		       VALUE
------------------------------ ----------------------------------------
DST_PRIMARY_TT_VERSION	       14
DST_SECONDARY_TT_VERSION       0
DST_UPGRADE_STATE	       NONE
```

- Prepare update timezone file version

Need to put upgrade mode to update the timezone file
```
SQL> shutdown immediate
SQL> startup upgrade
SQL> alter session set "_with_subquery"=materialize;
SQL> alter session set "_simple_view_merging"=TRUE;
SQL> exec DBMS_DST.BEGIN_PREPARE(18);
SQL> SELECT PROPERTY_NAME, SUBSTR(property_value, 1, 30) value
     FROM DATABASE_PROPERTIES WHERE PROPERTY_NAME LIKE 'DST_%' ORDER BY PROPERTY_NAME;

PROPERTY_NAME		       VALUE
------------------------------ ----------------------------------------
DST_PRIMARY_TT_VERSION	       14
DST_SECONDARY_TT_VERSION       18
DST_UPGRADE_STATE	       PREPARE

SQL> TRUNCATE TABLE SYS.DST$TRIGGER_TABLE;
SQL> TRUNCATE TABLE sys.dst$affected_tables;
SQL> TRUNCATE TABLE sys.dst$error_table;
SQL> set serveroutput on
SQL> BEGIN
     DBMS_DST.FIND_AFFECTED_TABLES
     (affected_tables => 'sys.dst$affected_tables',
     log_errors => TRUE,
     log_errors_table => 'sys.dst$error_table');
     END;
     /
#Below query should not return any rows
SQL> SELECT * FROM sys.dst$affected_tables;
SQL> SELECT * FROM sys.dst$error_table;
SQL> EXEC DBMS_DST.END_PREPARE;
A prepare window has been successfully ended.
```

- Begin update the timezone file

```
SQL> purge dba_recyclebin;
SQL> alter session set "_with_subquery"=materialize;
SQL> alter session set "_simple_view_merging"=TRUE;
SQL> EXEC DBMS_DST.BEGIN_UPGRADE(18);
SQL> SELECT PROPERTY_NAME, SUBSTR(property_value, 1, 30) value
FROM DATABASE_PROPERTIES WHERE PROPERTY_NAME LIKE 'DST_%' ORDER BY PROPERTY_NAME;
PROPERTY_NAME		       VALUE
------------------------------ ----------------------------------------
DST_PRIMARY_TT_VERSION	       18
DST_SECONDARY_TT_VERSION       14
DST_UPGRADE_STATE	       UPGRADE
#restart the instance
SQL> shutdown immediate
SQL>  startup
SQL> alter session set "_with_subquery"=materialize;
SQL> alter session set "_simple_view_merging"=TRUE;
SQL> set serveroutput on
     VAR numfail number
     BEGIN
     DBMS_DST.UPGRADE_DATABASE(:numfail,
     parallel => TRUE,
     log_errors => TRUE,
     log_errors_table => 'SYS.DST$ERROR_TABLE',
     log_triggers_table => 'SYS.DST$TRIGGER_TABLE',
     error_on_overlap_time => FALSE,
     error_on_nonexisting_time => FALSE);
     DBMS_OUTPUT.PUT_LINE('Failures:'|| :numfail);
     END;
/
SQL> VAR fail number
     BEGIN
     DBMS_DST.END_UPGRADE(:fail);
     DBMS_OUTPUT.PUT_LINE('Failures:'|| :fail);
     END;
/
```

- Verify the result

```
SQL> SELECT PROPERTY_NAME, SUBSTR(property_value, 1, 30) value
     FROM DATABASE_PROPERTIES WHERE PROPERTY_NAME LIKE 'DST_%' ORDER BY PROPERTY_NAME;
PROPERTY_NAME		       VALUE
------------------------------ ----------------------------------------
DST_PRIMARY_TT_VERSION	       18
DST_SECONDARY_TT_VERSION       0
DST_UPGRADE_STATE	       NONE

SQL> SELECT * FROM v$timezone_file;
FILENAME				    VERSION	CON_ID
---------------------------------------- ---------- ----------
timezlrg_18.dat 				 18	     0
```

- Update the registry

```
SQL> select TZ_VERSION from registry$database;
TZ_VERSION
----------
	14
SQL> update registry$database set TZ_VERSION = (select version FROM v$timezone_file);
SQL> commit;
SQL> select TZ_VERSION from registry$database;
TZ_VERSION
----------
	18
```

### 4. Convert non-CDB to PDB
When upgrade finished, the database in new 12c should be a non-CDB, which is DEPRECATED by Oracle. If the upgraded database is the only database on the current system, then you need to create a Container Database (CDB) first using dbca. Once the CDB is created you can proceed with the next step.
```
SQL> select name,open_mode,database_role,cdb from v$database;

NAME			       OPEN_MODE  DATABASE_ROLE 		   CDB
------------------------------ ---------- -------------------------------- ------
LINORA			       READ WRITE PRIMARY			   NO
```
#### 4.1 Create CDB
Please refer this post [Silent Install 12c](/oracle/12c-silent-installation.html) to build an empty CDB. My testing environment as below:
```
SQL> select con_id,dbid,name,open_mode,total_size from v$containers;

    CON_ID	 DBID NAME	 OPEN_MODE  TOTAL_SIZE
---------- ---------- ---------- ---------- ----------
	 1  285246148 CDB$ROOT	 READ WRITE	     0
	 2 1525007667 PDB$SEED	 READ ONLY   361758720
```
My CDB instance is ora12c, my non-CDB instance is linora.
#### 4.2 Convert non-CDB to PDB
When CDB is ready, there are ways can transfer the non-CDB to PDB: data pump, OGG and DBMS_PDB function. Recommend to use DBMS_PDB.

- Open non-CDB in read only mode

```
[oracle@linora:/home/oracle]$ export ORACLE_SID=linora
[oracle@linora:/home/oracle]$ sqlplus "/as sysdba"
SQL> shutdown immediate
SQL> startup mount exclusive
SQL> alter database open read only;
```

- Generate PDB xml manifest file

```
SQL> exec DBMS_PDB.DESCRIBE(pdb_descr_file => '/home/oracle/ncdb.xml');
SQL> shutdown immediate
```

- Connect to CDB, verify xml file

```
[oracle@linora:/home/oracle]$ export ORACLE_SID=ora12c
[oracle@linora:/home/oracle]$ sqlplus "/as sysdba"
SQL> SET SERVEROUTPUT ON
SET SERVEROUTPUT ON
DECLARE
hold_var boolean;
begin
hold_var := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(pdb_descr_file=>'/home/oracle/ncdb.xml');
if hold_var then
dbms_output.put_line('YES');
else
dbms_output.put_line('NO');
end if;
end;
/
YES
```
If the output is NO, please query the following view:
```
SQL> col cause for a25
col message for a85
set pagesize 9999
select cause,type,message,status from PDB_PLUG_IN_VIOLATIONS where name='LINORA';
CAUSE			  TYPE		     MESSAGE											STATUS
------------------------- ------------------ ------------------------------------------------------------------------------------------ ------------------
Parameter		  WARNING	     CDB parameter sga_target mismatch: Previous 564M Current 0 				RESOLVED
Service Name Conflict	  WARNING	     Service name or network name of service linora in the PDB is invalid or conflicts with an	RESOLVED
					     existing service name or network name in the CDB.

Parameter		  WARNING	     CDB parameter open_cursors mismatch: Previous 300 Current 500				RESOLVED
Parameter		  WARNING	     CDB parameter pga_aggregate_target mismatch: Previous 187M Current 10M			RESOLVED
Non-CDB to PDB		  ERROR 	     PDB plugged in is a non-CDB, requires noncdb_to_pdb.sql be run.				RESOLVED
Database CHARACTER SET	  ERROR 	     Character set mismatch: PDB character set WE8MSWIN1252. CDB character set AL32UTF8.	PENDING
OPTION			  ERROR 	     Database option OWM mismatch: PDB installed version 12.1.0.2.0. CDB installed version NULL PENDING
```

- Convert the non-CDB to PDB

```
SQL> CREATE PLUGGABLE DATABASE linora
USING '/home/oracle/ncdb.xml'
COPY
FILE_NAME_CONVERT = ('/oradata/linora/linora/',
'/data/PDB/linora/');
#Use NOCOPY, reuse tempfile
CREATE PLUGGABLE DATABASE linora USING '/home/oracle/ncdb.xml' NOCOPY TEMPFILE REUSE;
```

- Verify the result

```
SQL> select name, open_mode from v$pdbs;

NAME	   OPEN_MODE
---------- --------------------
PDB$SEED   READ ONLY
LINORA	   MOUNTED
SQL> col pdb_name for a15
select pdb_name, status from dba_pdbs;

PDB_NAME	STATUS
--------------- ------------------
PDB$SEED	NORMAL
LINORA		NEW
```

- Access to PDB

```
SQL> alter session set container=linora;
SQL> show con_name
CON_NAME
------------------------------
LINORA
#This script must be run before the new PDB can be opened for the first time
SQL> @?/rdbms/admin/noncdb_to_pdb.sql
```

- Access to CDB, verify the convert result

```
SQL> col cause for a25
SQL> col message for a85
SQL> set pagesize 9999
SQL> select cause,type,message,status from PDB_PLUG_IN_VIOLATIONS;

CAUSE                     TYPE               MESSAGE                                                                               STATUS
------------------------- ------------------ ------------------------------------------------------------------------------------- ------------------
Parameter                 WARNING            CDB parameter sga_target mismatch: Previous 3616M Current 3600M                       RESOLVED
Service Name Conflict     WARNING            Service name or network name of service orcl1 in the PDB is invalid or conflicts with RESOLVED
                                              an existing service name or network name in the CDB.

Parameter                 WARNING            CDB parameter compatible mismatch: Previous '12.1.0' Current '12.1.0.2.0'             RESOLVED
Parameter                 WARNING            CDB parameter pga_aggregate_target mismatch: Previous 1203M Current 1200M             RESOLVED
Non-CDB to PDB            ERROR              PDB plugged in is a non-CDB, requires noncdb_to_pdb.sql be run.                       RESOLVED
OPTION                    WARNING            Database option DV mismatch: PDB installed version NULL. CDB installed version 12.1.0 PENDING
                                             .2.0.

OPTION                    WARNING            Database option OLS mismatch: PDB installed version NULL. CDB installed version 12.1. PENDING
                                             0.2.0.
SQL> select con_id, name,open_mode from v$containers;

    CON_ID NAME                                                         OPEN_MODE
---------- ------------------------------------------------------------ --------------------
         1 CDB$ROOT                                                     READ WRITE
         2 PDB$SEED                                                     READ ONLY
         3 LINORA                                                       READ WRITE
```
You also can use v$database view to verify it:
```
SQL> select name, decode(cdb, 'YES', 'Multitenant Option enabled', 'Regular 12c Database: ') "Multitenant Option" , open_mode, con_id from v$database;
NAME		   Multitenant Option					OPEN_MODE      CON_ID
------------------ ---------------------------------------------------- ---------- ----------
ORA12C		   Multitenant Option enabled				READ WRITE	    0
```




<b>EOF</b></br>



