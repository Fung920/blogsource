---
layout: post
title: "Moving Files in database"
date: 2016-03-24 16:39:59
comments: false
categories: database
tags: database
keywords: move file 
description: how to move different types of files in oracle and DB2 database
---
As a production DBA, we may meet some file movement requirements, such as move a data file from a filesystem to another filesystem, move redo log files, move control files, even move standard file system to ASM. This topic discuss how to relocate those files in Oracle database and how to migrate the whole tablespace in DB2 database. 
<!--more-->
### 1. Relocating files in Oracle database
The most important file types in Oracle database are: data file, control file and redo log file, I'll show the most common approach to relocate these types of file. 
#### 1.1 Moving datafiles 
In oracle database 11g and earlier, moving datafiles will affect relevant applications or the whole database. As of Oracle database 12c, a new feature can let you moving the datafiles while the tablespace and the database keeping online. 

- Moving datafiles in 12c 

With 12c new feature, you can move any datafiles without any downtime, and can move all dafafiles in all tablespaces, including SYSTEM AND SYSAUX. It's also works with ASM diskgroup. 
```
SQL> select INSTANCE_NAME,VERSION,STATUS from v$instance;
INSTANCE_NAME			 VERSION			    STATUS
-------------------------------- ---------------------------------- ------------------------
linora				 12.1.0.2.0			    OPEN

SQL> select * from v$dbfile where file#=5;
     FILE# NAME 						  CON_ID
---------- -------------------------------------------------- ----------
	 5 /oradata/linora/fung01.dbf				       0

SQL> alter database move datafile '/oradata/linora/fung01.dbf' to '/data/linora/fung01.dbf';
--to ASM DG
SQL> alter database move datafile '/oradata/linora/fung01.dbf' to '+DATA';
SQL> select * from v$dbfile where file#=5;
     FILE# NAME 						  CON_ID
---------- -------------------------------------------------- ----------
	 5 /data/linora/fung01.dbf				       0
```
Previous post also can be referred in RAC or ASM environments: [RAC Datafile in Local Node](/rac-datafile-in-local-node.html) .

- Moving datafiles in 11g or earlier 

There are two approaches can accomplish this task: with <code>ALTER DATABASE</code> command or with <code>ALTER TABLESPACE</code> command. The difference between the two methods is that ALTER DATABASE need to bring the whole database down, and ALTER TABLESPACE only need to offline the relevant tablespaces. 
```
SQL> select INSTANCE_NAME,VERSION,STATUS from v$instance;
INSTANCE_NAME			 VERSION			    STATUS
-------------------------------- ---------------------------------- ------------------------
ora11g				 11.2.0.4.0			    OPEN

SQL> select * from v$dbfile;
     FILE# NAME
---------- --------------------------------------------------
	 1 /oradata/ora11g/ora11g/system01.dbf
	 2 /oradata/ora11g/ora11g/sysaux01.dbf
	 3 /oradata/ora11g/ora11g/undotbs01.dbf
	 4 /oradata/ora11g/ora11g/fung01.dbf
	 5 /oradata/ora11g/ora11g/users01.dbf
```

  - Moving datafiles with ALTER DATABASE 

This approach needs to shutdown the instance, because it needs to shutdown the instance, this method can move all the datafiles in all tablespaces including SYSTEM,SYSAUX etc., the general steps as below:
```
--moving fung01.dbf to /data/ora11g/ora11g
SQL> shutdown immediate
SQL> ! mv /oradata/ora11g/ora11g/fung01.dbf /data/ora11g/ora11g/ 
SQL> startup mount
SQL> alter database rename file '/oradata/ora11g/ora11g/fung01.dbf' to '/data/ora11g/ora11g/fung01.dbf';
SQL> alter database open;
SQL> select * from v$dbfile where file#=4;
     FILE# NAME
---------- --------------------------------------------------
	 4 /data/ora11g/ora11g/fung01.dbf
--perform an full backup includes controlfiles 
```

  - Moving datafiles with ALTER TABLESPACE

This approach cannot move SYSTEM,SYSAUX,active undo tablespace and temporary tablespace. With this method, you can keep your database online except the tablespace which datafiles will be moved.  The general steps are as follow:
```
SQL> alter tablespace fung offline;
SQL> !mv /data/ora11g/ora11g/fung01.dbf /oradata/ora11g/ora11g/fung01.dbf 
SQL> alter tablespace fung rename datafile '/data/ora11g/ora11g/fung01.dbf' to '/oradata/ora11g/ora11g/fung01.dbf';
SQL> alter tablespace fung online;
SQL> select * from v$dbfile where file#=4;
     FILE# NAME
---------- --------------------------------------------------
	 4 /oradata/ora11g/ora11g/fung01.dbf
```
#### 1.2 Moving the redo log files
Actually, you can just simply delete an entire redo log group and add a new redo log group in a different location [Manage Redo](/redo-manage.html) .This approach can be used if database needs to be kept open. Below approach is how to do it when the database shut down. 
```
SQL> select * from v$logfile;
    GROUP# STATUS	  TYPE		 MEMBER 					    IS_REC
---------- -------------- -------------- -------------------------------------------------- ------
	 1		  ONLINE	 /oradata/ora11g/ora11g/redo01.log		    NO
	 2		  ONLINE	 /oradata/ora11g/ora11g/redo02.log		    NO
	 3		  ONLINE	 /oradata/ora11g/ora11g/redo03.log		    NO

SQL> shutdown immediate
SQL> !mv /oradata/ora11g/ora11g/redo03.log /data/ora11g/ora11g/
SQL> startup mount
SQL> alter database rename file '/oradata/ora11g/ora11g/redo03.log' to '/data/ora11g/ora11g/redo03.log';
SQL> alter database open;
SQL> select * from v$logfile;
    GROUP# STATUS	  TYPE		 MEMBER 					    IS_REC
---------- -------------- -------------- -------------------------------------------------- ------
	 1		  ONLINE	 /oradata/ora11g/ora11g/redo01.log		    NO
	 2		  ONLINE	 /oradata/ora11g/ora11g/redo02.log		    NO
	 3		  ONLINE	 /data/ora11g/ora11g/redo03.log 		    NO
```
#### 1.3 Moving control files 
There are two approaches when moving control files depend on what initial files are you using. When using legacy init file instead of spfile, you need to shutdown the database, move the control file physically, and restart the database.   
Below method is about how to move the control file when using spfile.
```
SQL> select status,name from v$controlfile;
STATUS	       NAME
-------------- --------------------------------------------------
	       /oradata/ora11g/ora11g/control01.ctl
	       /oradata/ora11g/ora11g/control02.ctl
SQL> show parameter control_files 

NAME		     TYPE		    VALUE
------------------------- ---------------------- ------------------------------
control_files	     string	    /oradata/ora11g/ora11g/control01.ctl, /oradata/ora11g/ora11g/control02.ctl

SQL> alter system set control_files = '/data/ora11g/ora11g/control01.ctl',                                          
 '/data/ora11g/ora11g/control02.ctl' scope=spfile;
SQL> shutdown immediate
SQL> !mv /oradata/ora11g/ora11g/control01.ctl /data/ora11g/ora11g/control01.ctl
SQL> !mv /oradata/ora11g/ora11g/control02.ctl /data/ora11g/ora11g/control02.ctl
SQL> startup
SQL> show parameter control_files 
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
control_files			     string		    /data/ora11g/ora11g/control01.
							    ctl, /data/ora11g/ora11g/contr
							    ol02.ctl
SQL> select name from v$controlfile;
NAME
--------------------------------------------------
/data/ora11g/ora11g/control01.ctl
/data/ora11g/ora11g/control02.ctl
```

### 2. Migrate the whole tablespace in DB2 database
Depending on your storage configuration, there are different ways can let you move the tablespace to another location.  As you know, DB2 V10.1 provide <code>ALTER TABLESPACE... USING STOGROUP</code> can relocate the whole tablespace to another storage group when using AUTOMATIC STORAGE MANAGEMENT. Following examples shows how to migrate the tablespace with different methods in different circumstance. 
#### 2.1 Migrate tablespace in non-automatic storage 
There are three methods about moving tablespace in non-automatic storage.  Below is the original testing environment.
```
[db2inst1@node1 ~]$ db2pd -d testdb -tablespaces
Tablespace Configuration:
Address            Id    Type Content PageSz ExtentSz Auto Prefetch BufID BufIDDisk FSC NumCntrs MaxStripe  LastConsecPg RSE  Name
0x00007FBAE70EE560 4     DMS  Large   4096   32       Yes  64       1     1         Off 2        0          31           Yes  FUNG
Tablespace Autoresize Statistics:
Address            Id    AS  AR  InitSize             IncSize              IIP MaxSize              LastResize                 LRF
0x00007FBAE70EE560 4     No  No  -4096                0                    No  0                    None                       No 
#The AS field is No, mean non-automatic storage
Containers:
Address            TspId ContainNum Type    TotalPgs   UseablePgs PathID     StripeSet  Container 
Containers:
Address            TspId ContainNum Type    TotalPgs   UseablePgs PathID     StripeSet  Container 
0x00007FBAE70BE240 4     0          File    2560       2528       -          0          /data/db2inst1/NODE0000/SQL00001/fung01.LRG
0x00007FBAE70BE470 4     1          File    2560       2528       -          0          /data/db2inst1/NODE0000/SQL00001/fung02.LRG
```

I'd like to move the tablespace "FUNG" from /data/ to /data2/ file system by using different ways showing as below.  

- Adding the new containers without rebalance and delete the old containers 

First, add new containers to the target file system, I use <code>begin new stripe set</code>, this means no rebalancing occur when adding new containers, if you want to add containers to an exist stripe set, use <code> ADD TO STRIPE SET</code> clause. 
```
[db2inst1@node1 ~]$ db2 "alter tablespace fung begin new stripe set(file '/data2/db2inst1/NODE0000/SQL00001/fung01.LRG' 10M,
file '/data2/db2inst1/NODE0000/SQL00001/fung02.LRG' 10M)"

[db2inst1@node1 ~]$ db2pd -d testdb -tablespaces
Containers:
Address            TspId ContainNum Type    TotalPgs   UseablePgs PathID     StripeSet  Container 
0x00007FBAE70C3780 4     0          File    2560       2528       -          0          /data/db2inst1/NODE0000/SQL00001/fung01.LRG
0x00007FBAE70C39B0 4     1          File    2560       2528       -          0          /data/db2inst1/NODE0000/SQL00001/fung02.LRG
0x00007FBAE70C3BE0 4     2          File    2560       2528       -          1          /data2/db2inst1/NODE0000/SQL00001/fung01.LRG
0x00007FBAE70C3E10 4     3          File    2560       2528       -          1          /data2/db2inst1/NODE0000/SQL00001/fung02.LRG
```
Then, drop old containers, rebalancing will occur automatically in background. 
```
[db2inst1@node1 ~]$ db2 "alter tablespace fung drop ( file '/data/db2inst1/NODE0000/SQL00001/fung01.LRG', 
file '/data/db2inst1/NODE0000/SQL00001/fung02.LRG')"
```
We can use table function to monitor the rebalance status. 
```
--As of V10.1
db2 "
select 
   varchar(tbsp_name, 30) as tbsp_name, 
   dbpartitionnum, 
   member, 
   rebalancer_mode, 
   rebalancer_status, 
   rebalancer_extents_remaining, 
   rebalancer_extents_processed, 
   rebalancer_start_time 
from table(mon_get_rebalance_status(NULL,-2)) as t
"
--V9.7 and earlier
db2 "
select 
   varchar(tbsp_name, 30) as tbsp_name, 
   dbpartitionnum, 
   rebalancer_mode, 
   rebalancer_extents_remaining, 
   rebalancer_extents_processed, 
   rebalancer_start_time 
from table(snap_get_tbsp_part_v91(NULL,-2)) as t
"
```

- Performing a db2relocatedb utility 

This is a  much quickly way than adding new containers, if you have large data in this tablespace, the rebalancing will last very long time. But relocatedb can minimal the downtime via move physical file directly, this method need to restart the database.    

```
--configure the relocatedb parameter file as below
[db2inst1@node1 ~]$ cat relocatedb.cfg
DB_NAME=testdb
DB_PATH=/data/
INSTANCE=db2inst1
NODENUM=0
CONT_PATH=/data/db2inst1/NODE0000/SQL00001/fung01.LRG,/data2/db2inst1/NODE0000/SQL00001/fung01.LRG
CONT_PATH=/data/db2inst1/NODE0000/SQL00001/fung02.LRG,/data2/db2inst1/NODE0000/SQL00001/fung02.LRG
--deactivate the database, move the containers physically via OS command 
[db2inst1@node1 ~]$ db2 deactivate db testdb
[db2inst1@node1 ~]$ mv /data/db2inst1/NODE0000/SQL00001/fung01.LRG /data2/db2inst1/NODE0000/SQL00001/fung01.LRG
[db2inst1@node1 ~]$ mv /data/db2inst1/NODE0000/SQL00001/fung02.LRG /data2/db2inst1/NODE0000/SQL00001/fung02.LRG
--relocate the database 
[db2inst1@node1 ~]$ db2relocatedb -f relocatedb.cfg 
Files and control structures were changed successfully.
--activate the database, and verify the result
[db2inst1@node1 ~]$ db2pd -d testdb -tablespaces
...
Containers:
Address            TspId ContainNum Type    TotalPgs   UseablePgs PathID     StripeSet  Container 
0x00007FBADFA9A120 4     0          File    2560       2528       -          0          /data2/db2inst1/NODE0000/SQL00001/fung01.LRG
0x00007FBADFA9A350 4     1          File    2560       2528       -          0          /data2/db2inst1/NODE0000/SQL00001/fung02.LRG
```

- Redirect restoring the tablespace 

This is another optional way, backup images and the downtime are required. 
```
--generate redirect script
[db2inst1@node1 ~]$ db2 "restore db testdb REBUILD WITH all tablespaces in database from /data \
taken at 20160324193845 redirect generate script redirect.clp"
--modify the script of container 4 to
SET TABLESPACE CONTAINERS FOR 4
-- IGNORE ROLLFORWARD CONTAINER OPERATIONS
USING (
  FILE   '/data2/db2inst1/NODE0000/SQL00001/fung01.LRG'                    2560
, FILE   '/data2/db2inst1/NODE0000/SQL00001/fung02.LRG'                    2560
);
--executing redirect script
[db2inst1@node1 ~]$ db2 -tvf redirect.clp
--rollforward the db to end of logs 
[db2inst1@node1 ~]$ db2 rollforward db testdb to end of logs and complete
```
By verifying the result, you can find out the container path already changed. 

#### 2.2 Migrate tablespace in automatic storage 
It's very simple while relocating the tablespace to another location when using automatic storage, via ALTER TABLESPACE...USING STOGROUP you can simply specify the storage path which you want to move to.  You can also use <code>db2relocatedb</code>, but f you via db2relocatedb tool to relocate the containers, it's not support CONT_PATH, you need to reloate the whole storage group together. 
```
[db2inst1@node1 ~]$ db2pd -d source -storagepath|grep -i "Group Paths" -A 5
Storage Group Paths: 
Address            SGID  PathID    PathState    PathName
0x00007FBB1B348000 0     0         InUse        /db2/data/newstorage
[db2inst1@node1 ~]$ db2pd -d source -tablespaces
Tablespace Configuration:
Address            Id    Type Content PageSz ExtentSz Auto Prefetch BufID BufIDDisk FSC NumCntrs MaxStripe  LastConsecPg RSE  Name
0x00007FBB1E230080 2     DMS  Large   8192   32       Yes  64       1     1         Off 2        0          31           Yes  USERSPACE1
Containers:
Address            TspId ContainNum Type    TotalPgs   UseablePgs PathID     StripeSet  Container 
0x00007FBB226F9200 2     0          File    1920       1888       0          0          /db2/data/newstorage/db2inst1/NODE0000/SOURCE/T0000002/C0000002.LRG
```
Assumed my file system "/db2/data/newstorage" is encountered file system full issue, I'd like to move the whole tablespace "USERSPACE1" to another file system "/db2/data/db2inst1", only 2 steps can accomplish that, and with this method, no need to bring down database, the rebalance will start automatically and will move all tablespace data to the new path in the background.  
```
--create a storage path in /db2/data/db2inst1
[db2inst1@node1 ~]$ db2 "create stogroup sg2 on '/db2/data/db2inst1'"
--find out the create result
[db2inst1@node1 ~]$ db2  "select substr(SGNAME,1,20) as SGNAME,substr(OWNER,1,10) as owner, \
CREATE_TIME,DEFAULTSG from syscat.stogroups"
SGNAME               OWNER      CREATE_TIME                DEFAULTSG
-------------------- ---------- -------------------------- ---------
IBMSTOGROUP          SYSIBM     2016-03-22-14.03.00.472669 Y        
SG2                  DB2INST1   2016-03-24-17.38.22.583351 N        
--move the tablespace to new storage group
[db2inst1@node1 ~]$ db2 "alter tablespace USERSPACE1 using stogroup sg2"
[db2inst1@node1 ~]$ db2pd -d source -tablespaces
Tablespace Configuration:
Address            Id    Type Content PageSz ExtentSz Auto Prefetch BufID BufIDDisk FSC NumCntrs MaxStripe  LastConsecPg RSE  Name
0x00007FBB1E230080 2     DMS  Large   8192   32       Yes  32       1     1         Off 1        0          31           Yes  USERSPACE1
Containers:
Address            TspId ContainNum Type    TotalPgs   UseablePgs PathID     StripeSet  Container 
0x00007FBB226FBCE0 2     0          File    3776       3744       1024       0          /db2/data/db2inst1/db2inst1/NODE0000/SOURCE/T0000002/C0000004.LRG
```

#### 2.3. How to find out whether the tablespace is automatic storage or not 
Many ways can find out whether the databaspace is AS or not. 
```
db2 "select TBSP_USING_AUTO_STORAGE,tbsp_name from SYSIBMADM.SNAPTBSP"
db2 "select TBSP_USING_AUTO_STORAGE,tbsp_name from SYSIBMADM.TBSP_UTILIZATION"
db2 "select TBSP_USING_AUTO_STORAGE,tbsp_name from SYSIBMADM.MON_TBSP_UTILIZATION"
db2pd -db source -tablespaces #Tablespace Autoresize Statistics field "AS" value "YES"
db2 get snapshot for tablespaces on DB_NAME #Using automatic storage field value "YES" 
```



<b>EOF</b></br>



