---
title: DB2 Runstats and Reorgs
categories: db2
published: true
comments: false
date: 2017-08-22 17:44:09
tags:
---

 DB2 provide multiple tools and utilities for the maintenance, with these tools and utilities, it's more convenient for DBA to manage the DB2 database.
 <!--more-->

# 1. runstats and reorgs
<code>runstats</code> is for collecting indexes and tables statistics information which to enable the DB2 optimizer to generate efficient access plan.<code>reorgs</code> is for reorganizing tables and indexes.

## 1.1 runstats
Collect table and indexes information, including data distribution information:
```
db2 runstats on table db2inst1.employee on all columns with distribution and detailed indexes all
```

Collect indexes statistics information:
```
db2 runstats on table db2inst1.employee for indexes all
```

Collect table and indexes statistics information:
```
db2 runstats on table db2inst1.employee and detailed indexes all
```

Collect table statistics information with histogram distribution of column empid and empname, and give the table with read access:
```
db2 runstats on table db2inst1.employee with distribution on columns ( empid, empname ) allow read access
```

Verify if the tables have statistics or not:
```
[db2inst1@db2srv db2backup]$ db2 "select char(tabname,10) as tabname, stats_time from syscat.tables where tabname='EMPLOYEE'"
TABNAME    STATS_TIME
---------- --------------------------
EMPLOYEE   2017-08-22-17.20.12.999433
```

Scripts for generate the runstats:

```
#!/bin/bash
if [ "$#" < 3 ] ; then
echo "USAGE:$0 DB_NAME DB_USER_NAME DB_PASSWORD"
exit
fi
DB=$1
DB_USER=$2
DB_PWD=$3
db2 connect to $DB user $DB_USER using $DB_PWD
db2 "select trim('RUNSTATS ON TABLE ' || trim(tabschema) || '.' || tabname || '\
ON ALL COLUMNS WITH DISTRIBUTION ON ALL COLUMNS AND SAMPLED DETAILED \
INDEXES ALL ALLOW WRITE ACCESS;') from syscat.tables where type='T'" \
|grep RUNSTATS > runstats_detailed.sql
#db2 -tvf runstats_detailed.sql
```

## 1.2 reorgs and reorgchk
Use reorgchk to determine if a table/index need to be reorged or not.
```
db2 reorgchk on table db2inst1.employee
db2 reorgchk on schema db2inst1
```
if the column of reorgchk output F1~F3 marked as "*", it means the tables need to be reorged, if the F4~F8 columns marked as "*", it means the indexes need to be reorged

There have two different ways to reorgs, for 24*7 mission critical database, recommend the in-place reorgs, but it will generate lots of logs, and it can be terminated at any time; another way is called classic reorgs, with more fast and indexes will be built in more perfect order.

<center>Advantage and disadvantages for in-place and classic reorgs</center>

|              | In-place                                | Classic                      |
| :----         | ----                                    | ----                         |
| Advantages   | Allow applications access during reorgs | Fastest                      |
|              | Can be paused and resumed               | Index built in perfect order |
| Disadvantage | Imperfect indexes reorganization        | Large space required         |
|              | Longer time to complete                 | Limited table access         |
|              | Required more logs space                | All or nothing process


* Classic reorgs

The offline reorgs phases: 1. Sort, 2. Build, 3. Replace or copy, 4. Index rebuild


__Specify temporary tablespace__
```
db2 reorg table db2inst1.employee use TEMPSPACE1
```
__Use the original tablespace which the table reside in__
```
db2 reorg table db2inst1.employee
```

* In-place reorgs

```
db2 reorg table db2inst1.employee index i1 inplace allow write access
```

***Monitoring the reorgs***
```
[db2inst1@db2srv db2backup]$ db2 get snapshot for tables on testdb |grep -i employee -A 25
 Table Name          = EMPLOYEE
 Table Type          = User
 Data Object Pages   = 1
 Index Object Pages  = 6
 Rows Read           = Not Collected
 Rows Written        = 0
 Overflows           = 0
 Page Reorgs         = 0
 Table Reorg Information:
   Reorg Type        =
        Reclaiming
        Table Reorg
        Allow Read Access
        Reorg Data Only
   Reorg Index       = 0
   Reorg Tablespace  = 1
   Start Time        = 08/22/2017 19:21:03.312249
   Reorg Phase       = 3 - Index Recreate
   Max Phase         = 3
   Phase Start Time  = 08/22/2017 19:21:03.423726
   Status            = Completed
   Current Counter   = 0
   Max Counter       = 0
   Completion        = 0
   End Time          = 08/22/2017 19:21:03.527966
[db2inst1@db2srv db2backup]$ db2pd -d testdb -reorgs file=reorg.out
Sending -reorgs output to /db2backup/reorg.out.
[db2inst1@db2srv db2backup]$ db2 list history reorg all for testdb
db2 "select * from sysibmadm.snaptab_reorg"
db2 select * from table(sysproc.admin_list_hist( )) as listhistory
$HOME/sqllib/db2dump/<instance_name>
db2 "
SELECT SUBSTR(TABNAME, 1, 15) AS TAB_NAME,
SUBSTR(TABSCHEMA, 1, 15) AS TAB_SCHEMA,
REORG_PHASE, SUBSTR(REORG_TYPE, 1, 20) AS REORG_TYPE,
REORG_STATUS, REORG_COMPLETION, DBPARTITIONNUM
FROM SYSIBMADM.SNAPTAB_REORG ORDER BY DBPARTITIONNUM
"
```

***EOF***
