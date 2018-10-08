---
layout: post
title: "Recovering a dropped table in Oracle and DB2"
date: 2016-03-03 20:06:21
comments: false
categories: db2
tags: recover
keywords: recover dropped table
description: how to recover a dropped table in Oracle and DB2
---
When DBA dropped a table accidentally, always need a quick way to recover the dropped table. What I mean "quick way" is not only recovering the dropped table as soon as possible, but also should not affect other applications in the same database. 
<!--more-->
### 1. Recover a dropped table in Oracle 
If your database version is above 10g, and the recyclebin is enable, you can just simply use recyclebin feature to UNDO the dropped table. 
```
SQL> show parameter recyclebin
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
recyclebin			     string		    on
--drop a table
SQL> conn fung/oracle
Connected.
SQL> select count(*) from t;
  COUNT(*)
----------
     86260
SQL> drop table t;
Table dropped.
SQL> show recyclebin
ORIGINAL NAME	 RECYCLEBIN NAME		OBJECT TYPE  DROP TIME
---------------- ------------------------------ ------------ -------------------
T		 BIN$LSJDDhSaBtDgU8g4qMAzSA==$0 TABLE	     2016-03-03:17:02:45
SQL> flashback table t to before drop;
Flashback complete.
SQL> select count(*) from t;

  COUNT(*)
----------
     86260
```
#### 1.1 explanation of recyclebin
You can find more details in "recyclebin" view:
```
SQL> select object_name,original_name,operation,DROPTIME,CAN_UNDROP,CAN_PURGE from recyclebin;
OBJECT_NAME				 ORIGINAL_NAME	      OPERATION 	 DROPTIME				CAN_UN CAN_PU
------------------------- -------------------- ------------------ -------------------------------------- ------ ------
BIN$LSJDDhSbBtDgU8g4qMAzSA==$0		 T		      DROP		 2016-03-03:17:31:13			YES    YES
```
Recyclebin can be purged, if you don't want to keep the dropped in recyclebin, also can issue drop table command with purge options:
```
--purge recyclebin
SQL> purge recyclebin;
Recyclebin purged.
SQL> select * from recyclebin;
no rows selected
--drop table with purge option
SQL> create table t as select * from dba_objects;
Table created.
SQL> drop table t purge;
Table dropped.
SQL> show recyclebin;
```
Enable and disable recylebin feature:
```
--disable recyclebin, this step need to restart the instance
SQL> alter system set recyclebin=off scope=spfile;
System altered.
SQL> show parameter recyclebin
NAME				     TYPE		    VALUE
------------------- ---------------------- ------------------------------
recyclebin			     string		    OFF
--enable recyclebin, also need to restart the instance
SQL> alter system set recyclebin=on scope=spfile;
```
Multiple version in recyclebin, you may drop a table in couple of times, this action will generate multiple object with the same original name in recyclbin.
```
SQL> create table t as select * from dba_objects;
Table created.
SQL> drop table t;
Table dropped.
SQL> create table t as select * from dba_users;
Table created.
SQL> drop table t;
Table dropped.
SQL> create table t as select * from dba_tables;
Table created.
SQL> drop table t;
Table dropped.
SQL> show recyclebin
ORIGINAL NAME	 RECYCLEBIN NAME		OBJECT TYPE  DROP TIME
---------------- ------------------------------ ------------ -------------------
T		 BIN$LSLiyo5tB/LgU8g4qMDT/Q==$0 TABLE	     2016-03-03:17:47:52
T		 BIN$LSLiyo5qB/LgU8g4qMDT/Q==$0 TABLE	     2016-03-03:17:47:38
T		 BIN$LSLiyo5iB/LgU8g4qMDT/Q==$0 TABLE	     2016-03-03:17:47:25
```
As we can see, each dropped table T is assigned a unique name in the recyclebin, if you flashback the dropped table at that time, the most recently dropped table with that original name is retrieved from recyclebin, you also can indicate the unique name to specify the exactly table you want to recover.
```
--specified the unique name
SQL> flashback table "BIN$LSLiyo5iB/LgU8g4qMDT/Q==$0" to before drop;
Flashback complete.
```
Below action should be the most recently dropped table, which unique name is "BIN$LSLiyo5tB/LgU8g4qMDT/Q==$0", because we recover a same original name, you cannot recover it without indicate the "rename" option, or you should get "ORA-38312: original name is used by an existing object" error
```
SQL> flashback table t to before drop rename to t1;
Flashback complete.
SQL> show recyclebin
ORIGINAL NAME	 RECYCLEBIN NAME		OBJECT TYPE  DROP TIME
---------------- ------------------------------ ------------ -------------------
T		 BIN$LSLiyo5qB/LgU8g4qMDT/Q==$0 TABLE	     2016-03-03:17:47:38
```
### 2. Recover a dropped table in DB2 
Recovery dropped in DB2 is somewhat more complex than in Oracle. In Oracle 10g and above, if you enable recyclebin feature(default enabled), you can easily take recovery without any downtime, no rman recovery need.    
But in DB2, you need a backup image to accomplish this task. With backup image, restore the tablespace which the dropped table reside on, and import the data from recover data. Here's an example.
```
[db2v97i@db2srv ~]$ db2 "create table fung.t(id varchar(10),name varchar(20)) in fung"
DB20000I  The SQL command completed successfully.
[db2v97i@db2srv ~]$ db2 "insert into fung.t values(51369,'fung')"
DB20000I  The SQL command completed successfully.
[db2v97i@db2srv ~]$ db2 terminate
DB20000I  The TERMINATE command completed successfully.
[db2v97i@db2srv ~]$ db2 deactivate db mysample
DB20000I  The DEACTIVATE DATABASE command completed successfully.

--take a offline backup
[db2v97i@db2srv ~]$ db2 backup db mysample to /db2/backup/db2v97i/mysample/ compress
Backup successful. The timestamp for this backup image is : 20160307161004

--drop the table T 
[db2v97i@db2srv ~]$ db2 drop table fung.t
DB20000I  The SQL command completed successfully.
```
First, try to find out whether the tablespace support "DROPPED TABLE RECOVERY" or not 
```
[db2v97i@db2srv ~]$ db2 "select substr(TBSPACE,1,20) as TBS,TBSPACETYPE,DROP_RECOVERY from SYSCAT.TABLESPACES"
TBS                  TBSPACETYPE DROP_RECOVERY
-------------------- ----------- -------------
SYSCATSPACE          D           N            
TEMPSPACE1           S           N            
USERSPACE1           D           Y            
IBMDB2SAMPLEREL      D           Y            
SYSTOOLSPACE         D           Y            
SYSTOOLSTMPSPACE     S           N            
FUNG                 D           Y  `
```
Second, find out the dropped table 
```
[db2v97i@db2srv ~]$ db2 list history dropped table all for mysample
                    List History File for mysample
 Op Obj Timestamp+Sequence Type Dev Earliest Log Current Log  Backup ID
 -- --- ------------------ ---- --- ------------ ------------ --------------
  D  T  20160307161043                                        000000000000f6b800060004 
 ----------------------------------------------------------------------------
  "FUNG    "."T" resides in 1 tablespace(s):

 00001 FUNG
 ----------------------------------------------------------------------------
    Comment: DROP TABLE
 Start Time: 20160307161043
   End Time: 20160307161043
     Status: A
 ----------------------------------------------------------------------------
  EID: 604

 DDL: CREATE TABLE "FUNG    "."T" ( "ID" VARCHAR(10) , "NAME" VARCHAR(20) )  IN "FUNG" ;   
 ----------------------------------------------------------------------------
```
From the output, we can retrieve the dropped table name, which tablespace did the table reside on, and the DDL of the table.   
Third, restore a database-level or tablespace-level backup image taken before the table was dropped.
```
--find out available backup image 
[db2v97i@db2srv ~]$ db2 list history backup all for mysample
                    List History File for mysample
 Op Obj Timestamp+Sequence Type Dev Earliest Log Current Log  Backup ID
 -- --- ------------------ ---- --- ------------ ------------ --------------
  B  D  20160307161004001   F    D  S0000380.LOG S0000380.LOG  
 ----------------------------------------------------------------------------
  Contains 5 tablespace(s):

 00001 SYSCATSPACE
 00002 USERSPACE1
 00003 IBMDB2SAMPLEREL
 00004 SYSTOOLSPACE
 00005 FUNG
 ----------------------------------------------------------------------------
    Comment: DB2 BACKUP MYSAMPLE OFFLINE
 Start Time: 20160307161004
   End Time: 20160307161012
     Status: A
 ----------------------------------------------------------------------------
  EID: 602 Location: /db2/backup/db2v97i/mysample

--execute the restore command
[db2v97i@db2srv ~]$ db2 "restore db mysample tablespace (FUNG) 
online from /db2/backup/db2v97i/mysample/ taken at 20160307161004"
DB20000I  The RESTORE DATABASE command completed successfully.
```
Next, create an export directory to which files contaning the table data are to be written, and rollforward to a PIT. 
```
[db2v97i@db2srv ~]$ mkdir -p /restore
[db2v97i@db2srv ~]$ db2 "ROLLFORWARD DB mysample TO END OF LOGS TABLESPACE 
ONLINE RECOVER DROPPED TABLE 000000000000f6b800060004 TO /restore"
                                 Rollforward Status
 Input database alias                   = mysample
 Number of members have returned status = 1
 Member ID                              = 0
 Rollforward status                     = not pending
 Next log file to be read               =
 Log files processed                    =  -
 Last committed transaction             = 2016-03-07-08.10.43.000000 UTC
DB20000I  The ROLLFORWARD command completed successfully.
```
Next, re-create the dropped table (DDL can obtain from list history command ).
```
db2 => CREATE TABLE "FUNG    "."T" ( "ID" VARCHAR(10) , "NAME" VARCHAR(20) )  IN "FUNG"
DB20000I  The SQL command completed successfully.
```
Finally, we can import the data from restore directory.
```
--import the data from that was export during the roll forward operation into the table
[db2v97i@db2srv ~]$ db2 "IMPORT FROM /restore/NODE0000/data OF DEL INSERT INTO FUNG.T"
SQL3109N  The utility is beginning to load data from file "/restore/NODE0000/data".
SQL3110N  The utility has completed processing.  "1" rows were read from the input file.
SQL3221W  ...Begin COMMIT WORK. Input Record Count = "1".
SQL3222W  ...COMMIT of any database changes was successful.
SQL3149N  "1" rows were processed from the input file.  
	  "1" rows were successfully inserted into the table.  "0" rows were rejected.

Number of rows read         = 1
Number of rows skipped      = 0
Number of rows inserted     = 1
Number of rows updated      = 0
Number of rows rejected     = 0
Number of rows committed    = 1

[db2v97i@db2srv ~]$ db2 "select * from fung.t"
ID         NAME                
---------- --------------------
51369      fung                
  1 record(s) selected.
```

### 3. Supplemental 
There's still one mistaken operation that always happens. Delete table records happens frequently, maybe you delete some records, or you delete all the table records. So, what's the difference between DB2 and Oracle database while recovering "deleted records"?
#### 3.1 recovering deleted records in Oracle
It's very simple to use flashback query feature to undo the deleted data in tables. If you didn't commit the delete command, you also can rollback the delete statement easily.
```
SQL> select count(*) from fung.t;
  COUNT(*)
----------
     86260
SQL> !date
Mon Mar  7 19:24:40 CST 2016
```
Now, let's delete table data and commit it.
```
SQL> delete from fung.t;
86260 rows deleted.
SQL> commit;
Commit complete.
```
Undo the delete operation by using flashback query feature:
```
SQL> select count(*) from fung.t as of timestamp to_timestamp('2016-03-07 19:24:00','yyyy-mm-dd hh24:mi:ss');
  COUNT(*)
----------
     86260
SQL> insert into fung.t select * from fung.t as of timestamp to_timestamp('2016-03-07 19:24:00','yyyy-mm-dd hh24:mi:ss');
86260 rows created.
SQL> commit;
Commit complete.
SQL> select count(*) from fung.t;
  COUNT(*)
----------
     86260
```
Data is back. If you don't know what the exactly deletion time, you can adjust the timestamp until find the proper one. Flashback query feature is based on undo tablespace, if undo tablespace or the parameter <code>undo_retention</code> (value by seconds) not big or long enough, you also may need take a RMAN recovery to undo the detetion. 
```
SQL> show parameter undo_retention
NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
undo_retention			     integer		    900
```
#### 3.2 recovering deleted records in DB2
DB2 recovery technology is always more complex than Oracle. Oracle can just use flashback feature to recover deleted records. And auto-commit is enable in DB2 by default, if you didn't disable auto-commit, you cann't even use rollback to undo the delete command.    
```
[db2ace@oc7421025535 ~]$ db2 list command options |grep -i commit
   -c    Auto-Commit                               ON
```
In DB2, the best way to recover a deleted table is redirect restore. Let's simulate a circumstance to explain it.
```
--duplicate a system table structure 
[db2v97i@db2srv ~]$ db2 "create table fung.t1 like syscat.tables in FUNG"
DB20000I  The SQL command completed successfully.
--insert some records into the table 
[db2v97i@db2srv ~]$ db2 "insert into fung.t1 select * from syscat.tables"
DB20000I  The SQL command completed successfully.
[db2v97i@db2srv ~]$ db2 "select count(*) from fung.t1"
1          
-----------
        502
  1 record(s) selected.
```
I didn't take any backup after I create this table. I use the previous backup in step 2. 
```
[db2v97i@db2srv ~]$ date
Mon Mar  7 17:57:00 CST 2016
--delete all the records 
[db2v97i@db2srv ~]$ db2 "delete from fung.t1"
DB20000I  The SQL command completed successfully.
[db2v97i@db2srv ~]$ db2 "select count(*) from fung.t1"
1          
-----------
          0
  1 record(s) selected.
```
Generate the redirect restore command, I don't want to pollute current production environment. So I decide restore the DB/Tablespace in another place. 
```
[db2v97i@db2srv ~]$ db2 "restore db mysample rebuild with tablespace (SYSCATSPACE,FUNG) 
from /db2/backup/db2v97i/mysample/ taken at 20160307161004 redirect generate script redirect.clp"
DB20000I  The RESTORE DATABASE command completed successfully.
```
Edit the redirect.clp file to proper values, modify the DBPATH, LOGPATH and the DBNAME, and execute the restore operation. 
```
--change the DBPATH to /restore/mysample
[db2v97i@db2srv ~]$ mkdir -p /restore/mysample
--change the log path to /restore/mysample/log
[db2v97i@db2srv ~]$ mkdir -p /restore/mysample/log
```
After I edited the redirect.clp file, it should like below.
```
[db2v97i@db2srv ~]$ grep -v ^- redirect.clp 
UPDATE COMMAND OPTIONS USING S ON Z ON MYSAMPLE_NODE0000.out V ON;
SET CLIENT ATTACH_MEMBER  0;
SET CLIENT CONNECT_MEMBER 0;
RESTORE DATABASE MYSAMPLE
REBUILD WITH TABLESPACE (
  "SYSCATSPACE"
, "TEMPSPACE1"
, "SYSTOOLSTMPSPACE"
, "FUNG"
)
FROM '/db2/backup/db2v97i/mysample/'
TAKEN AT 20160307161004
ON '/restore/mysample'
INTO NEWDB
NEWLOGPATH '/restore/mysample/log'
REDIRECT
;
RESTORE DATABASE MYSAMPLE CONTINUE;
```
Redirect the database into NEWDB with rebuild tablespace option:
```
[db2v97i@db2srv ~]$ db2 -tvf redirect.clp 
UPDATE COMMAND OPTIONS USING S ON Z ON MYSAMPLE_NODE0000.out V ON
DB20000I  The UPDATE COMMAND OPTIONS command completed successfully.

SET CLIENT ATTACH_MEMBER  0
DB20000I  The SET CLIENT command completed successfully.

SET CLIENT CONNECT_MEMBER 0
DB20000I  The SET CLIENT command completed successfully.

RESTORE DATABASE MYSAMPLE REBUILD WITH TABLESPACE ( "SYSCATSPACE" , "TEMPSPACE1" , "SYSTOOLSTMPSPACE" , "FUNG" ) FROM '/db2/backup/db2v97i/mysample/' TAKEN AT 20160307161004 ON '/restore/mysample' INTO NEWDB NEWLOGPATH '/restore/mysample/log' REDIRECT
SQL1277W  A redirected restore operation is being performed. During a table space restore, only table spaces being restored can have their paths 
reconfigured. During a database restore, storage group storage paths and DMS 
table space containers can be reconfigured.
DB20000I  The RESTORE DATABASE command completed successfully.

RESTORE DATABASE MYSAMPLE CONTINUE
DB20000I  The RESTORE DATABASE command completed successfully.
```
Find out which logs are needed by rollforward operation, and copy those logs file into the new log path:
```
[db2v97i@db2srv ~]$ db2 rollforward database newdb QUERY STATUS
                                 Rollforward Status
 Input database alias                   = newdb
 Number of members have returned status = 1
 Member ID                              = 0
 Rollforward status                     = DB  pending
 Next log file to be read               = S0000380.LOG
 Log files processed                    =  -
 Last committed transaction             = 2016-03-07-08.10.12.000000 UTC

[db2v97i@db2srv ~]$ db2 get db cfg |grep -i log
 Path to log files                                       = /db2/log/db2v97i/mysample/NODE0000/LOGSTREAM0000/
 First active log file                                   = S0000385.LOG
 First log archive method                 (LOGARCHMETH1) = DISK:/db2/arch/mysample/

[db2v97i@db2srv ~]$ find /db2 -name "S0000380.LOG"
/db2/arch/mysample/db2v97i/MYSAMPLE/NODE0000/LOGSTREAM0000/C0000001/S0000380.LOG
[db2v97i@db2srv ~]$ cp /db2/arch/mysample/db2v97i/MYSAMPLE/NODE0000/LOGSTREAM0000/C0000001/S000038*.LOG \
/restore/mysample/log/NODE0000/LOGSTREAM0000/
[db2v97i@db2srv ~]$ cp /db2/log/db2v97i/mysample/NODE0000/LOGSTREAM0000/S0000385.LOG \
/restore/mysample/log/NODE0000/LOGSTREAM0000/
[db2v97i@db2srv ~]$ db2 "rollforward database newdb to 2016-03-07-17.57.00.000000 USING LOCAL TIME and stop"
SQL1271W  Database "NEWDB" is recovered but one or more table spaces are 
offline on members or nodes "0".

[db2v97i@db2srv ~]$ db2 connect to newdb
   Database Connection Information
 Database server        = DB2/LINUXX8664 10.1.5
 SQL authorization ID   = DB2V97I
 Local database alias   = NEWDB

[db2v97i@db2srv ~]$ db2 "select count(*) from fung.t1"
1          
-----------
        502
  1 record(s) selected.
```
And now, you can export the data from the redirect db to the source db:
```
--export from newdb
[db2v97i@db2srv ~]$ db2 "export to t1.ixf of ixf select * from fung.t1 "
SQL3132W  The character data in column "STATISTICS_PROFILE" will be truncated to size "32700".
SQL3104N  The Export utility is beginning to export data to file "t1.ixf".
SQL3105N  The Export utility has finished exporting "502" rows.
Number of rows exported: 502

--import to sourcedb
[db2v97i@db2srv ~]$ db2 terminate
DB20000I  The TERMINATE command completed successfully.

[db2v97i@db2srv ~]$ db2 connect to mysample
   Database Connection Information
 Database server        = DB2/LINUXX8664 10.1.5
 SQL authorization ID   = DB2V97I
 Local database alias   = MYSAMPLE
[db2v97i@db2srv ~]$ db2 "select count(*) from fung.t1"
1          
-----------
          0
  1 record(s) selected.

[db2v97i@db2srv ~]$ db2 "import from t1.ixf of ixf allow write access insert into fung.t1"
SQL3150N  The H record in the PC/IXF file has product "DB2    02.00", date "20160307", and time "190757".
SQL3153N  The T record in the PC/IXF file has name "t1.ixf", qualifier "", and source "            ".
SQL3109N  The utility is beginning to load data from file "t1.ixf".
SQL3110N  The utility has completed processing.  "502" rows were read from the input file.
SQL3221W  ...Begin COMMIT WORK. Input Record Count = "502".
SQL3222W  ...COMMIT of any database changes was successful.
SQL3149N  "502" rows were processed from the input file.  
	  "502" rows were successfully inserted into the table.  "0" rows were rejected.

Number of rows read         = 502
Number of rows skipped      = 0
Number of rows inserted     = 502
Number of rows updated      = 0
Number of rows rejected     = 0
Number of rows committed    = 502

[db2v97i@db2srv ~]$ db2 "select count(*) from fung.t1"
1          
-----------
        502
  1 record(s) selected.
```
### 4. summary
Recover a deleted table is very simple in Oracle, but you need recover it as soon as possible, because depends on your production environment, no one knows how long will the undo data retain in the undo tablespace. So when you created your database, modify the parameter "undo_retention" to meet your business requirments.   
In DB2, the "dropped table recovery" feature do not support export XML column, so when recovering this type of data, PIT recovery should be a better choice. Besides, import will generate lots of logs and may have big impact of performance, when loading large tables into the source db, db2 LOAD utilities will be more suitable. 

<b>EOF</b></br>



