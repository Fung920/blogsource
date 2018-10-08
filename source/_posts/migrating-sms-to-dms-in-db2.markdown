---
layout: post
title: "Migrating SMS to DMS in DB2"
date: 2016-10-18 07:12:16
comments: false
categories: db2
tags:
keywords: db2look, db2move
description: How to use db2look and db2move to migrate SMS to DMS
name: How to use db2look and db2move to migrate SMS to DMS
author: Fung Kong
datePublished: 2016-10-18 07:12:16
---
Some DBA would be required to convert the SMS tablespace to DMS tablespace. But seems DB2 do not have such utilities to convert SMS to DMS directly. We cannot use backup/restore for restoring the SMS to DMS, the traditional way to convert SMS to DMS is to use `db2look` and `db2move`. Also hope some people can tell me a better option.
<!--more-->
To convert the SMS to DMS( in my production environment, there are over 300 tables belong to multiple schemas), following steps would be required(Outage required):

```
1. Extract tables' DDL in the tablespace   
2. Export all the data belong to the tablespace   
3. Create a new DMS tablespace   
4. Drop the old SMS tablespace   
5. Modify the DDL script and create the tables from the DDL script   
6. Load the data back into the new DMS tablespace   
7. Set integrity for the tables
```

## 1. Extracting the DDL of the tables
Before migration:

```sql
[db2inst1@db2srv ~]$ db2 "select substr(TABLE_SCHEMA,1,10) as schema,substr(TABLE_NAME,1,15) as tabname, \
substr(TABLESPACE_NAME,1,15) as tbsname from sysibmadm.dba_all_tables where TABLESPACE_NAME='CONVERT1'"

SCHEMA     TABNAME         TBSNAME        
---------- --------------- ---------------
FUNG       CL_SCHED        CONVERT1       
FUNG       EMPLOYEE        CONVERT1       
FUNG       DEPARTMENT      CONVERT1       
FUNG       EMP_PHOTO       CONVERT1       
FUNG       EMP_RESUME      CONVERT1       
FUNG       PROJECT         CONVERT1       
FUNG       PROJACT         CONVERT1       
FUNG       EMPPROJACT      CONVERT1       
FUNG       ACT             CONVERT1       
FUNG       IN_TRAY         CONVERT1       
FUNG       ORG             CONVERT1       
FUNG       STAFF           CONVERT1       
FUNG       SALES           CONVERT1       
FUNG       STAFFG          CONVERT1       
FUNG       ADEFUSR         CONVERT1       
DB2INST1   EMPLOYEE        CONVERT1       
DB2INST1   CL_SCHED        CONVERT1       
DB2INST1   DEPARTMENT      CONVERT1       
DB2INST1   PROJACT         CONVERT1       
DB2INST1   EMPPROJACT      CONVERT1       
DB2INST1   ACT             CONVERT1       
DB2INST1   EMP_PHOTO       CONVERT1       
DB2INST1   EMP_RESUME      CONVERT1       
DB2INST1   PROJECT         CONVERT1       
DB2INST1   IN_TRAY         CONVERT1       
DB2INST1   ORG             CONVERT1       
DB2INST1   STAFF           CONVERT1       
DB2INST1   SALES           CONVERT1       
DB2INST1   STAFFG          CONVERT1       
DB2INST1   ADEFUSR         CONVERT1       
```
There're two options which can let you extract the table structure.

### Option A: Extracting the whole schema's table DDl
From the output, there are only two schemas in my SMS table space, if we have fewer related schemas, this option can be considered.

```bash
[db2inst1@db2srv ~]$ db2look -d testdb -z FUNG -e -o fung_schema.sql
-- No userid was specified, db2look tries to use Environment variable USER
-- USER is: DB2INST1
-- Specified SCHEMA is: FUNG
-- Creating DDL for table(s)

-- Schema name is ignored for the Federated Section
-- Output is sent to file: fung_schema.sql
[db2inst1@db2srv ~]$ db2look -d testdb -z DB2INST1 -e -o db2inst1_schema.sql
-- No userid was specified, db2look tries to use Environment variable USER
-- USER is: DB2INST1
-- Specified SCHEMA is: DB2INST1
-- Creating DDL for table(s)

-- Schema name is ignored for the Federated Section
-- Output is sent to file: db2inst1_schema.sql
```

Now the table  definition are saved to the output files.
### Option B: Extracting the table DDL only in the tablespace
This method would be recommended if we have many schemas reside in the SMS tablespace.

```bash
#Generating extract DDL statement
db2 "select 'db2look -d testdb -t '|| rtrim(TABSCHEMA) ||'.'||'\"' \
||TABNAME||'\"'|| ' -e' from syscat.tables where TBSPACEID='2'" > db2look.sql
#Generating the DDL script
./db2look.sql >o.txt
#Remove the commit state, otherwise, only the first DDL would be executed
sed -i '/^COMMIT\ WORK/d' o.txt
sed -i '/^CONNECT\ RESET/d' o.txt
sed -i '/^TERMINATE/d' o.txt
```

## 2. Exporting all the tables reside in the tablespace
Export the table data into a directory by using db2move, db move will generate two files for each table, so it's a good idea to put all the files into a separate directory.

```bash
[db2inst1@db2srv ~]$ mkdir -p /db2backup/db2move; cd /db2backup/db2move/
[db2inst1@db2srv db2move]$ db2move testdb export -ts convert1
```

## 3. Dropping the old tablespace and creating a new tablespace

```bash
[db2inst1@db2srv db2move]$ db2 drop tablespace convert1
[db2inst1@db2srv db2move]$ db2 "create tablespace DMSTBS1 managed by database using \
(file '/db/db2inst1/testdb/db2inst1/NODE0000/TESTDB/T0000005/myfile' 2048, \
file '/db/db2inst1/testdb/db2inst1/NODE0000/TESTDB/T0000005/myfile2' 2048) extentsize 4"
```

## 4. Modifying the DDL script and creating the tables from the DDL script

```bash
[db2inst1@db2srv ~]$ sed -i 's/CONVERT1/DMSTBS1/g' fung_schema.sql
[db2inst1@db2srv ~]$ sed -i 's/CONVERT1/DMSTBS1/g' db2inst1_schema.sql
```
Double check the script to ensure there's no missing anything. If everything is OK, then creating the table into the new tablespace:

```bash
[db2inst1@db2srv ~]$ db2 connect to testdb
[db2inst1@db2srv ~]$ db2 -tvf fung_schema.sql
[db2inst1@db2srv ~]$ db2 -tvf db2inst1_schema.sql
```
Double check to ensure everything is fine, then we can proceed.

## 5. Loading the data back into the new tablespace
```bash
[db2inst1@db2srv ~]$ cd /db2backup/db2move/
[db2inst1@db2srv db2move]$ db2move testdb load -l ./
```
Compare the output of LOAD.out and EXPORT.out, if the committed rows are exactly equal, then everything is okay.

## 6. Setting the integrity for the tables
First, using following SQL to query the table status:
```sql
Select substr(tabschema,1,8) as "Qualified Name",
substr(tabname,1,50) as "Table name",
CASE type
WHEN 'A' THEN 'Alias'
WHEN 'H' THEN 'Hierarchy Table'
WHEN 'N' THEN 'Nickname'
WHEN 'S' THEN 'Summary Table'
WHEN 'T' THEN 'Table'
WHEN 'U' THEN 'Typed Table'
WHEN 'V' THEN 'View'
WHEN 'W' THEN 'Typed View'
END as "Table Type",
CASE status
WHEN 'N' THEN 'Normal'
WHEN 'C' THEN 'Check Pending'
WHEN 'X' THEN 'Inoperative'
END as "Table Status"
from syscat.tables;
[db2inst1@db2srv ~]$ db2 -tvf finding_all_table_status.sql |grep -v Normal

Qualified Name Table name                                         Table Type      Table Status 
-------------- -------------------------------------------------- --------------- -------------
FUNG           DEPARTMENT                                         Table           Check Pending
FUNG           EMPLOYEE                                           Table           Check Pending
FUNG           EMP_PHOTO                                          Table           Check Pending
```

Generating the `set integrity` command:
```bash
[db2inst1@db2srv ~]$ db2 "select 'set integrity for '|| rtrim(tabschema)||'.'||tabname|| ' \
IMMEDIATE CHECKED;' from syscat.tables where status !='N'">setint.  sql
[db2inst1@db2srv ~]$ db2 -tvf setint.sql
```
Some errors are expected, because we separated co-dependency tables to set integrity, which they should be together.

```bash
set integrity for FUNG.EMPLOYEE IMMEDIATE CHECKED
DB21034E  The command was processed as an SQL statement because it was not a 
valid Command Line Processor command.  During SQL processing it returned:
SQL3608N  Cannot check a dependent table "FUNG.EMPLOYEE" using the SET 
INTEGRITY statement while the parent table or underlying table 
"FUNG.DEPARTMENT" is in the Set Integrity Pending state or if it will be put 
into the Set Integrity Pending state by the SET INTEGRITY statement.  
SQLSTATE=428A8
```

Execute the `set integrity` together for the co-dependency tables:
```bash
SET INTEGRITY FOR <table1>, <table2> IMMEDIATE CHECKED
[db2inst1@db2srv ~]$ db2 set schema fung
DB20000I  The SQL command completed successfully.
[db2inst1@db2srv ~]$ db2 set integrity for EMPLOYEE,EMP_PHOTO,EMP_RESUME, \
PROJECT,PROJACT,EMPPROJACT,ADEFUSR IMMEDIATE CHECKED
DB20000I  The SQL command completed successfully.
```

Finally, we're here, converted the SMS to DMS succefully.

***EOF***
