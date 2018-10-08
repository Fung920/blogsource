---
layout: post
title: "Migrate non-ASM to ASM in 12c"
date: 2016-04-01 16:12:09
comments: false
categories: oracle
tags: 12c
keywords: ASM,Standalone 
description: Migrate non-ASM to ASM
---
There are many approaches can convert file system to ASM, such as RMAn copy database image, as of 12c, you can move datafiles online, that feature enables you minimize the downtime. 
<!--more-->
### 1. Abstract 
My CDB named ora12c contain one PDB named pdb1 with local file system datafiles. I have my GI standalone installed with diffenert owner GRID contains two asm disk groups, I need to migrate the whole CDB to ASM. Below is the current environment before migration. 

```
#GI already installed with 2 disk groups 
[grid@linora:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.dg
               ONLINE  ONLINE       linora                   STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       linora                   STABLE
ora.asm
               ONLINE  ONLINE       linora                   Started,STABLE
ora.ons
               OFFLINE OFFLINE      linora                   STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       linora                   STABLE
ora.diskmon
      1        OFFLINE OFFLINE                               STABLE
ora.evmd
      1        ONLINE  ONLINE       linora                   STABLE
--------------------------------------------------------------------------------
SQL> select NAME,STATE from v$asm_diskgroup;
NAME			       STATE
------------------------------ -----------
FRA			       MOUNTED
DATA			       MOUNTED
```

### 2. Find out the datafiles to be converted 
Connect to CDB ROOT container via RMAN, use <code>report schema</code> which can show you all the datafiles and tempfiles that inlcuding PDBs and CDB.
```
RMAN> report schema;
using target database control file instead of recovery catalog
Report of database schema for database with db_unique_name ORA12C
List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    790      SYSTEM               YES     /oradata/ora12c/system01.dbf
2    260      PDB$SEED:SYSTEM      NO      /oradata/ora12c/pdbseed/system01.dbf
3    690      SYSAUX               NO      /oradata/ora12c/sysaux01.dbf
4    595      PDB$SEED:SYSAUX      NO      /oradata/ora12c/pdbseed/sysaux01.dbf
5    1480     UNDOTBS1             YES     /oradata/ora12c/undotbs01.dbf
6    5        USERS                NO      /oradata/ora12c/users01.dbf
7    260      PDB1:SYSTEM          NO      /oradata/ora12c/pdb1/system01.dbf
8    605      PDB1:SYSAUX          NO      /oradata/ora12c/pdb1/sysaux01.dbf
9    5        PDB1:USERS           NO      /oradata/ora12c/pdb1/pdb1_users01.dbf
10   100      PDB1:FUNG            NO      /oradata/ora12c/pdb1/fung01.dbf
List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1    67       TEMP                 32767       /oradata/ora12c/temp01.dbf
2    62       PDB$SEED:TEMP        32767       /oradata/ora12c/pdbseed/temp01.dbf
3    20       PDB1:TEMP            32767       /oradata/ora12c/pdb1/temp01.dbf
```

### 3. Moving the datafiles online to ASM disk group 
```
SQL> set lines 200
set pages 50
set feed off
set head off
spool /home/oracle/move_dbfiles.sql
select 'ALTER DATABASE MOVE DATAFILE '''||name||''' TO ''+DATA'';' from v$datafile order by con_id;
spool off
```
Because there are multiple PDBs in the CDB, I need to change the session to the PDB container. I adjust the spooled file like below:
```
[oracle@linora:/home/oracle]$ cat move_dbfiles.sql 
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/system01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/undotbs01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/users01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/sysaux01.dbf' TO '+DATA';
ALTER SESSION SET CONTAINER=pdb$seed;
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/pdbseed/sysaux01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/pdbseed/system01.dbf' TO '+DATA';
ALTER SESSION SET CONTAINER=pdb1;
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/pdb1/fung01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/pdb1/pdb1_users01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/pdb1/system01.dbf' TO '+DATA';
ALTER DATABASE MOVE DATAFILE '/oradata/ora12c/pdb1/sysaux01.dbf' TO '+DATA';
```
### 4. Moving the tempfile to ASM disk group 
We can add the new tempfile and drop the old one to migrate the tempfile. 
```
#Moving CDB tempfiles
SQL> alter tablespace TEMP add tempfile '+DATA';
SQL> alter tablespace TEMP drop tempfile '/oradata/ora12c/temp01.dbf';
#Moving PDB tempfiles
SQL> alter session set container=pdb1;
SQL> alter tablespace TEMP add tempfile '+DATA';
SQL>  alter tablespace TEMP drop tempfile '/oradata/ora12c/pdb1/temp01.dbf';
#Moving PDB$SEED tempfiles 
SQL> alter session set container=CDB$ROOT;
SQL> alter session set "_oracle_script"=TRUE;
SQL> alter pluggable database pdb$seed close;
SQL> alter pluggable database pdb$seed open read write;
SQL> alter session set container=pdb$seed;
SQL> alter tablespace temp add tempfile '+DATA';
SQL> alter tablespace temp drop tempfile '/oradata/ora12c/pdbseed/temp01.dbf';
SQL> alter session set container=CDB$ROOT;
SQL> alter pluggable database pdb$seed close;
SQL> alter pluggable database pdb$seed open read only;
```
Query the modified result in RMAN:
```
RMAN> report schema;
using target database control file instead of recovery catalog
Report of database schema for database with db_unique_name ORA12C
List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    790      SYSTEM               YES     +DATA/ORA12C/DATAFILE/system.257.908042629
2    260      PDB$SEED:SYSTEM      NO      +DATA/ORA12C/2F64B9185F472450E0534638A8C061D3/DATAFILE/system.262.908042885
3    690      SYSAUX               NO      +DATA/ORA12C/DATAFILE/sysaux.260.908042811
4    595      PDB$SEED:SYSAUX      NO      +DATA/ORA12C/2F64B9185F472450E0534638A8C061D3/DATAFILE/sysaux.261.908042849
5    1480     UNDOTBS1             YES     +DATA/ORA12C/DATAFILE/undotbs1.258.908042721
6    5        USERS                NO      +DATA/ORA12C/DATAFILE/users.259.908042809
7    260      PDB1:SYSTEM          NO      +DATA/ORA12C/2F659BBCFD612D6DE0534638A8C0AEF9/DATAFILE/system.265.908042909
8    605      PDB1:SYSAUX          NO      +DATA/ORA12C/2F659BBCFD612D6DE0534638A8C0AEF9/DATAFILE/sysaux.266.908042925
9    5        PDB1:USERS           NO      +DATA/ORA12C/2F659BBCFD612D6DE0534638A8C0AEF9/DATAFILE/users.264.908042907
10   100      PDB1:FUNG            NO      +DATA/ORA12C/2F659BBCFD612D6DE0534638A8C0AEF9/DATAFILE/fung.263.908042901
List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
4    100      TEMP                 32767       +DATA/ORA12C/TEMPFILE/temp.267.908047665
5    100      PDB1:TEMP            32767       +DATA/ORA12C/2F659BBCFD612D6DE0534638A8C0AEF9/TEMPFILE/temp.268.908047759
6    100      PDB$SEED:TEMP        32767       +DATA/ORA12C/2F64B9185F472450E0534638A8C061D3/TEMPFILE/temp.269.908047875
```
### 5. Moving redo log files 
Redo log files only exisit in CDB, because all PDBs share the same redo, so when crash recovery needed, only can CDB do it. 
```
SQL> select member from v$logfile;
MEMBER
--------------------------------------------------------------------------------
/oradata/ora12c/redo01.log
/oradata/ora12c/redo02.log
/oradata/ora12c/redo03.log
SQL> select group#, status from v$log;

    GROUP# STATUS
---------- --------------------------------
	 1 INACTIVE
	 2 INACTIVE
	 3 CURRENT
```
I dropped a group member and re-create it in ASM disk group to migrate the redo log files. 
```
SQL> alter database drop logfile group 1;
SQL> alter database add logfile group 1 '+DATA';
SQL> alter database drop logfile group 2;
SQL> alter database add logfile group 2 '+DATA';
#Log group 3 is current log, switch it first 
SQL> alter system switch logfile;
SQL> alter system checkpoint;
SQL> alter database drop logfile group 3;
SQL>  alter database add logfile group 3 '+DATA';
```
Query the result from the dynamic view:
```
SQL> select member from v$logfile;
MEMBER
--------------------------------------------------------------------------------
+DATA/ORA12C/ONLINELOG/group_1.270.908048777
+DATA/ORA12C/ONLINELOG/group_2.271.908048801
+DATA/ORA12C/ONLINELOG/group_3.272.908048857
SQL> select group#, status from v$log;
    GROUP# STATUS
---------- --------------------------------
	 1 CURRENT
	 2 UNUSED
	 3 UNUSED
```
### 6. Moving control files and spfile to ASM disk group
Activate the control files new location needs to bring down the database, this is the only step which need to downtime in non-ASM to ASM migration in 12c. 
```
SQL> show parameter control_files
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
control_files			     string		    /oradata/ora12c/control01.ctl,
							     /FRA/ora12c/control02.ctl
```
Shutdown the instance, bring the database in nomount mode, restore current controfiles to ASM, and modify the location of controlfiles in spfile/initial file. 
```
SQL> shutdown immediate
SQL> startup nomount;
#Restoring the current control files to ASM via RMAN
RMAN> restore controlfile to '+DATA' from '/oradata/ora12c/control01.ctl';
RMAN> restore controlfile to '+FRA' from '/FRA/ora12c/control02.ctl';
#Query the control files restored to ASM via asmcmd
ASMCMD [+] > find --type CONTROLFILE +DATA *
+DATA/ORA12C/CONTROLFILE/current.273.908049269
ASMCMD [+] > find --type CONTROLFILE +FRA *
+FRA/ORA12C/CONTROLFILE/current.256.908049303
#Modify the controlfiles location in spfile
SQL> alter system set control_files='+DATA/ORA12C/CONTROLFILE/current.273.908049269','+FRA/ORA12C/CONTROLFILE/current.256.908049303' scope=spfile;
#Change the FRA to ASM
SQL> alter system set db_recovery_file_dest='+FRA' scope=spfile;
#Modify archive log destination to FRA 
SQL> alter system reset log_archive_dest_1;
#Modify local listener
SQL> alter system set local_listener='(ADDRESS=(PROTOCOL=TCP)(HOST=LINORA)(PORT=1522))' scope=spfile;
```
Now, the spfile still in file system, I need to move it to ASM, and create a legacy initial file point to the spfile location in ASM. 
```
#Restoring the spfile to ASM via RMAN
RMAN> alter database mount ;
RMAN> run
{
BACKUP AS BACKUPSET SPFILE;
RESTORE SPFILE TO "+DATA/ORA12C/spfileora12c.ora";
}
```
Rename or remove the old initial file or spfile in $ORACLE_HOME/dbs, edit a new initial file with below contents:
```
#delete the old spfile/initial file 
[oracle@linora:/u01/app/oracle/product/12.0.1/db1/dbs]$ rm -rf initora12c.ora 
[oracle@linora:/u01/app/oracle/product/12.0.1/db1/dbs]$ rm -rf spfileora12c.ora
[oracle@linora:/u01/app/oracle/product/12.0.1/db1/dbs]$ cat >>initora12c.ora<<EOF
> SPFILE='+DATA/ORA12C/spfileora12c.ora'
> EOF
[oracle@linora:/u01/app/oracle/product/12.0.1/db1/dbs]$ cat initora12c.ora 
SPFILE='+DATA/ORA12C/spfileora12c.ora'
```
Start the database with the new spfile in ASM:
```
SQL> shutdown immediate
SQL> startup
SQL> show parameter pfile 
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
spfile				     string		    +DATA/ORA12C/spfileora12c.ora
SQL> show parameter control_files
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
control_files			     string		    +DATA/ORA12C/CONTROLFILE/curre
							    nt.273.908049269, +FRA/ORA12C/
							    CONTROLFILE/current.256.908049
							    303
```
### 7. Post-migration tasks 
The rest tasks will be add the service to the standalone grid infrastructure, including the database, the listener. 
#### 7.1 Adding the database resource to GI 
```
#Add the database resource
[oracle@linora:/home/oracle]$ srvctl add database -d ora12c -o $ORACLE_HOME -startoption OPEN -policy AUTOMATIC -v
#Add the spfile to new location 
[oracle@linora:/home/oracle]$ srvctl modify database -db ora12c -o $ORACLE_HOME -spfile '+DATA/ORA12C/spfileora12c.ora' -diskgroup "DATA,FRA"
```
#### 7.2 Adding the database listener resource to GI 
After you installed and created GI instance, the listener will create automatically, but it cannot listen the database. The database listener cannot manage by the GI too. This step is listener migration from database to GI.

- Remove the GI listener resource  

```
[grid@linora:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.dg
               ONLINE  ONLINE       linora                   STABLE
ora.FRA.dg
               ONLINE  ONLINE       linora                   STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       linora                   STABLE
ora.asm
               ONLINE  ONLINE       linora                   Started,STABLE
ora.ons
               OFFLINE OFFLINE      linora                   STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       linora                   STABLE
ora.diskmon
      1        OFFLINE OFFLINE                               STABLE
ora.evmd
      1        ONLINE  ONLINE       linora                   STABLE
ora.ora12c.db
      1        ONLINE  ONLINE       linora                   Open,STABLE
--------------------------------------------------------------------------------
[grid@linora:/home/grid]$ srvctl stop listener -listener listener
[grid@linora:/home/grid]$ srvctl remove listener -all
[grid@linora:/home/grid]$ pgrep -lf tns
15 netns
2442 /u01/app/oracle/product/12.0.1/db1/bin/tnslsnr LISTENER -inherit
```

- Copy the database listener  configuration to GI HOME 

```
[oracle@linora:/home/oracle]$ lsnrctl stop
[grid@linora:/home/grid]$ cd $ORACLE_HOME/network/admin/
[grid@linora:/u02/app/grid/12.1.0/grid/network/admin]$ cp /u01/app/oracle/product/12.0.1/db1/network/admin/listener.ora ./
[grid@linora:/u02/app/grid/12.1.0/grid/network/admin]$ cp /u01/app/oracle/product/12.0.1/db1/network/admin/tnsnames.ora ./
[grid@linora:/u02/app/grid/12.1.0/grid/network/admin]$ cp /u01/app/oracle/product/12.0.1/db1/network/admin/sqlnet.ora ./
[grid@linora:/u02/app/grid/12.1.0/grid/network/admin]$ cat >> listener.ora <<EOF 
ENABLE_GLOBAL_DYNAMIC_ENDPOINT_LISTENER=ON
EOF
[grid@linora:/u02/app/grid/12.1.0/grid/network/admin]$ sed -i '/ADR_BASE_LISTENER/s/\/u01\/app\/oracle/\/u02\/app\/grid/g' listener.ora
```

- Add the new listener to GI 

```
[grid@linora:/home/grid]$ srvctl add listener -l LISTENER -p "TCP:1522" -o $ORACLE_HOME
[grid@linora:/home/grid]$ srvctl start listener -listener LISTENER
[grid@linora:/home/grid]$ pgrep -lf tns
15 netns
7429 /u02/app/grid/12.1.0/grid/bin/tnslsnr LISTENER -no_crs_notify -inherit
[grid@linora:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.dg
               ONLINE  ONLINE       linora                   STABLE
ora.FRA.dg
               ONLINE  ONLINE       linora                   STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       linora                   STABLE
ora.asm
               ONLINE  ONLINE       linora                   Started,STABLE
ora.ons
               OFFLINE OFFLINE      linora                   STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       linora                   STABLE
ora.diskmon
      1        OFFLINE OFFLINE                               STABLE
ora.evmd
      1        ONLINE  ONLINE       linora                   STABLE
ora.ora12c.db
      1        ONLINE  ONLINE       linora                   Open,STABLE

```
Connect to the database via TCP/IP, if possible, reboot the OS to see if the database can autostart with the GI or not. 

### 8. Trouble shooting 
When I tried to start the database, encountered the below errors:
```
ORA-15025: could not open disk "/dev/asm-diskb"
ORA-27041: unable to open file
Linux-x86_64 Error: 13: Permission denied
Additional information: 3
```
Solutions:
```
[root@linora ~]# su - grid
[grid@linora:/home/grid]$ cd $ORACLE_HOME/bin
[grid@linora:/u02/app/grid/12.1.0/grid/bin]$  ./setasmgidwrap o=/u01/app/oracle/product/12.0.1/db1/bin/oracle
```

If you need the database autostart with the GI, don't forget to add dba group to grid user:
```
[grid@linora:/home/grid]$ id
uid=501(grid) gid=500(oinstall) groups=500(oinstall),501(dba),502(asmadmin),503(asmdba)
```



<b>EOF</b></br>

