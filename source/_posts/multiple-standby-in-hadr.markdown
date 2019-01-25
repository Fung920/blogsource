---
layout: post
title: "Multiple Standbys in HADR"
date: 2016-03-21 19:26:22
comments: false
categories: db2
tags: hadr
keywords: Multiple standby
description: multiple standby in HADR
---
Before DB2 V10.1, HADR only support one standby server. But from V10.1, multiple standby can be supported up to three standby databases. This feature truelly combine the HA and DR together. In this post, I'd like to show you how to initialize a multiple standbys system and how to rolling update through multiple standbys.
<!--more-->
### 1. Introduction
Some terminologies or parameters have been introduced for multiple standby.

- hadr_target_list

Initializing a HADR with multiple standby is similar to single standby mode. The main difference between single and multiple standby is the parameter <code>hadr_target_list</code>, this parameter should be set on all participate databases. This parameter's value specified on the primary determines how many standbys does the primary has.

- Principle standby

The first standby in multiple standby mode is called principle standby, it support most of the HADR features, RoS, delayed replay, takeover ect.. And additional, all of synchronization modes are supported in priciple standby.

- Auxiliary standby

The other standbys except principle standby can be called auxiliary standby, both types of standbys support RoS, delayed replay, takeover. But auxiliary standbys only support supersync mode.

### 2. Initialize a mutilple standby
There are 2 approaches to initialize multiple standbys.

- Initialize a totally new multiple standby HADR

- Convert a single standby HADR to a multiple standby HADR

Actually, it's not so much difference between these 2 approaches, for a single standby, just need to add <code>hadr_target_list</code> value for it become a mutilple standbys. In this post, I'll introduce how to setup a totally new multiple standby HADR. Below is my testing environment.

```
HOSTNAME	|INSTANCE	|DBNAME		|SVCENAME	|ROLE
node1		db2inst1	sample		51000		Primary
node2		db2inst1	sample		51001		Principle
node3		db2inst1	sample		51002		Auxiliary
```
To initialize multiple standby in HADR, complete the following steps.
#### 2.1 Configure required database parameters in primary
Configure archival log mode if archive not enable, configure LOGINDEXBUILD.
```
DBPATH=/db2/data/$INSTANCE
ACTLOG=/db2/log/$INSTANCE
BKPATH=/db2/backup/$INSTANCE
ARCLOG=/db2/arch/$INSTANCE
DBGROUP=db2adm
DBNAME=sample
mkdir -p $ACTLOG/$DBNAME
mkdir -p $ARCLOG/$DBNAME
mkdir -p $BKPATH/$DBNAME
db2 update db cfg for $DBNAME using NEWLOGPATH $ACTLOG/$DBNAME
db2 update db cfg for $DBNAME using LOGARCHMETH1 disk:$ARCLOG/$DBNAME
db2 update db cfg for $DBNAME using LOGINDEXBUILD ON
db2 terminate
db2 deactivate database $DBNAME
db2 update db cfg for $DBNAME using trackmod on
db2 backup db $DBNAME to /dev/null
db2 activate database $DBNAME
```
#### 2.2 Backup primary database
Perform a full offline/online backup of primary, copy the backup images to all the standbys.
```
[db2inst1@node1 ~]$ db2 backup db sample online to /db2/backup/db2inst1/ compress
Backup successful. The timestamp for this backup image is : 20160321150828
[db2inst1@node1 ~]$ scp /db2/backup/db2inst1/*20160321150828* node2:/db2/backup/db2inst1/
[db2inst1@node1 ~]$ scp /db2/backup/db2inst1/*20160321150828* node3:/db2/backup/db2inst1/
```
#### 2.3 Restore the database on all standbys
```
[db2inst1@node2 ~]$ db2 restore db sample from /db2/backup/db2inst1/ taken at 20160321150828  DBPATH ON '/db2/data/db2inst1' redirect
[db2inst1@node2 ~]$ db2 restore db sample continue
[db2inst1@node3 ~]$ db2 restore db sample from /db2/backup/db2inst1/ taken at 20160321150828  DBPATH ON '/db2/data/db2inst1' redirect
[db2inst1@node3 ~]$ db2 restore db sample continue
```
#### 2.4 Add the following entries in the service file
Add the HADR service name to /etc/services for all of the servers to setup HADR communication. This service name is different from database service name.
```
Service name: hadr_node1_db1
Port:		51000/tcp

Service name: hadr_node2_db1
Port:		51001/tcp

Service name: hadr_node3_db1
Port:		51002/tcp
```
#### 2.5 Update database configuration on primary node1
Update the HADR configuration parameter on priamry
```
[db2inst1@node1 ~]$ db2 "
update db cfg for sample using
HADR_LOCAL_HOST node1
HADR_LOCAL_SVC hadr_node1_db1
HADR_REMOTE_HOST node2
HADR_REMOTE_SVC hadr_node2_db1
HADR_REMOTE_INST db2inst1
HADR_TARGET_LIST node2:51001|node3:51002
HADR_SYNCMODE NEARSYNC
"
db2 connect to SAMPLE
db2 quiesce database immediate force connections
db2 unquiesce database
db2 connect reset
```
#### 2.6 Update database configuration on principle node2
```
[db2inst1@node2 ~]$ db2 "
update db cfg for sample using
HADR_LOCAL_HOST node2
HADR_LOCAL_SVC hadr_node2_db1
HADR_REMOTE_HOST node1
HADR_REMOTE_SVC hadr_node1_db1
HADR_REMOTE_INST db2inst1
HADR_TARGET_LIST node1:51000|node3:51002
HADR_SYNCMODE NEARSYNC
"
DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
```
#### 2.7 Update database configuration on auxilary node3
```
[db2inst1@node3 ~]$ db2 "
update db cfg for sample using
HADR_LOCAL_HOST node3
HADR_LOCAL_SVC hadr_node3_db1
HADR_REMOTE_HOST node1
HADR_REMOTE_SVC hadr_node1_db1
HADR_REMOTE_INST db2inst1
HADR_TARGET_LIST node2:51001|node1:51000
HADR_SYNCMODE superasync
"
DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
```
#### 2.8 Start the HADR
First, start HADR standby in node2 and node3.
```
[db2inst1@node2 ~]$ db2 deactivate database SAMPLE
[db2inst1@node2 ~]$ db2 start hadr on database SAMPLE as standby

[db2inst1@node3 ~]$ db2 deactivate database SAMPLE
[db2inst1@node3 ~]$ db2 start hadr on database SAMPLE as standby
```
Second, start HADR on primary node1.
```
[db2inst1@node1 ~]$ db2 deactivate database SAMPLE
[db2inst1@node1 ~]$ db2 start hadr on database SAMPLE as primary
```
To verify the HADR is running, run the <code>db2pd -d sample -hadr</code> command to monitor the HADR status, or use <code>mon_get_hadr</code> table function to monitor HADR status:
```
[db2inst1@node1 ~]$ db2 "select HADR_ROLE, STANDBY_ID, HADR_STATE,
             varchar(PRIMARY_MEMBER_HOST,20) as PRIMARY_HOST,
             varchar(STANDBY_MEMBER_HOST,20) as STANDBY_HOST
      from table (mon_get_hadr(NULL))"

HADR_ROLE     STANDBY_ID HADR_STATE              PRIMARY_HOST         STANDBY_HOST
------------- ---------- ----------------------- -------------------- --------------------
PRIMARY                1 PEER                    node1                node2
PRIMARY                2 REMOTE_CATCHUP          node1                node3
```
### 3. Rolling update in multiple standbys
HADR can provide High Avaliability while performing some maintenance tasks such as fix pack with minimal downtime or without any downtime. I have introduced how to rolling fix pack in single standby [Rolling Update and Upgrade in HADR](/db2/rolling-update-and-upgrade-in-hadr.html) , and we can take a look at how to rolling update in multiple standbys.
The generate steps of updating as follow:

- Deactivate node3, make any required changes, activate node3, and start node3 as standby
- Repeate above tasks in node2
- Takeover from node2, node2 now should be the primary
- Deactivate node1, upgrade/update in node1, start node1 as standby
- Takeover from node1 as primary, post-installation tasks

Below demo is how to update from V10.1 to V10.1_FP5 with minimal downtime. We can install DB2 V10.1_FP5 binary code in all three servers in a new directory beforehand to minimize downtime. So I assumed all the V10.1_FP5 code have been installed into /opt/ibm/db2/V10.1_FP5 directory beforehand.
```
[db2inst1@node1 ~]$ db2ls
Install Path                       Level   Fix Pack   Special Install Number   Install Date                  Installer UID
---------------------------------------------------------------------------------------------------------------------
/opt/ibm/db2/V10.1               10.1.0.0        0                            Thu Mar 17 14:30:52 2016 CST             0
/opt/ibm/db2/V10.1_FP5           10.1.0.5        5                            Thu Mar 17 16:24:37 2016 CST             0
```
#### 3.1 Rolling update (fix pack) in multiple standby

- Deactivate node3, make any required changes, activate node3, and start node3 as standby

```
[db2inst1@node3 ~]$ db2 deactivate db sample
[db2inst1@node3 ~]$ db2stop
[root@node3 worktmp]# /opt/ibm/db2/V10.1_FP5/instance/db2iupdt db2inst1
[root@node3 worktmp]# su - db2inst1
[db2inst1@node3 ~]$ db2licm -a /worktmp/db2ese_c.lic
[db2inst1@node3 ~]$ db2start
[db2inst1@node3 ~]$ db2 activate db sample
```

- Repeate above tasks in node2

```
[db2inst1@node2 ~]$ db2 deactivate db sample
[db2inst1@node2 ~]$ db2stop
[root@node2 worktmp]# /opt/ibm/db2/V10.1_FP5/instance/db2iupdt db2inst1
[root@node2 worktmp]# su - db2inst1
[db2inst1@node2 ~]$ db2licm -a /worktmp/db2ese_c.lic
[db2inst1@node2 ~]$ db2start
[db2inst1@node2 ~]$ db2 activate db sample
```

- Takeover from node2, node2 now should be the primary

```
[db2inst1@node2 ~]$ db2 takeover hadr on db sample
```

- Deactivate node1, upgrade/update in node1, start node1 as standby

```
[db2inst1@node1 ~]$ db2 deactivate db sample
[db2inst1@node1 ~]$ db2stop
[root@node1 worktmp]# /opt/ibm/db2/V10.1_FP5/instance/db2iupdt db2inst1
[root@node1 worktmp]# su - db2inst1
[db2inst1@node1 ~]$ db2licm -a /worktmp/db2ese_c.lic
[db2inst1@node1 ~]$ db2start
[db2inst1@node1 ~]$ db2 activate db sample
```

- Takeover from node1 as primary, post-installation tasks

```
[db2inst1@node1 ~]$ db2 takeover hadr on db sample
[db2inst1@node1 ~]$ /opt/ibm/db2/V10.1_FP5/bin/db2updv10 -d sample
[db2inst1@node1 ~]$ db2 connect to sample
[db2inst1@node1 ~]$ db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/db2schema.bnd BLOCKING ALL GRANT PUBLIC SQLERROR CONTINUE
[db2inst1@node1 ~]$ db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/@db2ubind.lst BLOCKING ALL GRANT PUBLIC ACTION ADD
[db2inst1@node1 ~]$ db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/@db2cli.lst BLOCKING ALL GRANT PUBLIC ACTION ADD
[db2inst1@node1 ~]$ db2rbind sample -l ./rbind.log all
[db2inst1@node1 ~]$ db2pd -v
Instance db2inst1 uses 64 bits and DB2 code release SQL10015 with level identifier 0206010E
Informational tokens are DB2 v10.1.0.5, s150624, IP23772, Fix Pack 5.
```

</br>
<b>EOF</b>


