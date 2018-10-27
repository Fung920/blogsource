---
layout: post
title: "HADR in DB2"
date: 2016-03-09 17:36:03
comments: false
categories: db2
tags: hadr
keywords: HADR
description: A Briefly Introduction of HADR in DB2
---
HADR is an abbrevation of High Avaiablity Disaster Recovery, as it's names implies, HADR is a software solution providing high avaiablity and disaster recovery for databases, expecially for 24*7 mission critical systems. The technology can be classified as a replication solution. Basically, it replicates data by sending and replaying logs from a source database called primary database to a target database called standby database. HADR is currently avaliable for single-partitioned databases only.
<!--more-->
### 1. Introduction
#### 1.1 Requirements
<li>OS should be the same version including patches. If rolling update happening, you can violate this rules for a short time</li>
<li>DB2 version and the level must be identical on both servers, including the bit size. (32bit or 64bit)</li>
<li>Nerwork interface must be available on both server</li>
<li>Bufferpool size should be the same</li>
<li>The database name should be identical, that means they must be in different instances if they reside in a same server</li>
<li>Tablespace must be identical</li>
<li>The log file space also should be the same on both server</li>
<li>The system clock must be sychronized on both servers</li>
#### 1.2 Drawback of HADR
<li>Backup operations are not supported on standby databases</li>
<li>Not support infinite logging</li>
<li>Not support no logging transaction</li>
<li>Not support for DPF</li>
<li>Not support in DB2 pureScale environments (v10.5 support)</li>
#### 1.3 What HADR can do
<li>Read on standby (supported db2 v9.7.1 and above)</li>
Read on standby can enable you to run read-only operations on standby.
<li>Rolling update without any downtime for running applications</li>
Enable you rolling update the database with minimal downtime or zero downtime when updating a fix pack version, but not supported upgrade from major version to a high major version, for example, it's supported update from 9.7.5 to 9.7.10, but not supported upgrade from 9.7 to 10.1. The main reason that HADR rolling update cannot cross major version boundary is that DB2 transaction logs from the new release may not be compatible with the old release, but HADR requires log compatibility on primary and standby.
<li>Delayed reply, new feature in 10.1</li>
This feature helps prevent data loss due to errant transactions. When standby server set the <code>hadr_replay_delay</code> database configuration parameter,it will keep standby database at a point in time that is earlier than primary database, if someone deleted some database by accidentally in primary, because standby didn't replay this errant transaction, so you can only copy the deleted data from standby to primary, or just take over in standby as primary.
<li>Log Spooling, new feature in 10.1</li>
This feature allows transactions on primary to make progress without having to wait for the log replay on the standby. When this feature is enabled, log data sent by the primary is spooled, or written, to disk on the standby, and that log data is later read by log replay.
<li>Mutilple standby, new feature in 10.1</li>
Before 10.1, HADR only supported one standby database, but from 10.1 and above, it's supported multiple standby database. For example, you can deploy a standby for delayed replying, and another standby for normal purpose.
### 2. Setup HADR with command line
Summary of testing information:
```
server name 		|instance name 			|hadr port				|database name
hadr01  		  db2v10i 			  DB2_HADR01_P/51000				mysample
hadr02		  db2v10i			  DB2_HADR02_S/51001
```
#### 2.1 Prepare for the environment
If it's a totally new build env, you can install and create DB2 database as [Install DB2 in Linux](/db2/install-db2-in-linux.html). As in standby server, only install software and create instance, no need to create database.  Also you need an identical env as primary, so do not forget to create necessery directories in standby server, set permissions on the new directories.
#### 2.2 Setup the requirement of primary
<li>turn on the archival log mode</li>
```
db2 update db cfg for mysample using LOGARCHMETH1 disk:/db2/arch/mysample
```
<li>enable LOGINDEXBUILD</li>
Set the parameter <code>LOGINDEXBUILD</code> to ON so that index creation, recreation, or reorgnization operations are logged. If the parameter is set to OFF, then there's not enough logging information for building the indexes on the standby, therefore, the new indexes created or rebuilt are marked as invalid on the standby. Also will increase time consume when standby activate because of rebuilding the invalid indexes.
```
db2 update database configuration for mysample using LOGINDEXBUILD ON
```
<li>configure INDEXREC parameater</li>
This parameter controls the rebuild of invalid indexes on database startup, in HADR, this parameter should be set to RESTART on both servers.
```
db2 update dbm cfg using INDEXREC RESTART
```
#### 2.3 Take an offline backup of primary
Take an offline backup and send the backup image to the standby, restore to rollforward pending status, DO NOT issue rollforward command. Because standby database needs in rollforward pending state for replaying logs.
```
--on primary
db2 backup db mysample to /db2/backup/
scp /db2/backup/MYSAMPLE.0.db2v10i.NODE0000.CATN0000.20160308233246.001 hadr02:/db2/backup/
--on standby
[db2v10i@hadr02 ~]$ mkdir -p /db2/arch/mysample
[db2v10i@hadr02 ~]$ mkdir -p /db2/data/db2v10i
[db2v10i@hadr02 ~]$ mkdir -p /db2/log/mysample
[db2v10i@hadr02 ~]$ db2 restore db mysample from /db2/backup taken at 20160308233246 DBPATH ON '/db2/data/db2v10i' redirect
[db2v10@node2 ~]$ db2 restore db mysample continue
--do not forget set LOGINDEXBUILD to ON in standby server
db2 update database configuration for mysample using LOGINDEXBUILD ON
```
#### 2.4 Configure database parameters of HADR on primary
```
db2 update db cfg for mysample using HADR_LOCAL_HOST hadr01 	--should be primary ip addr or hostname
db2 update db cfg for mysample using HADR_LOCAL_SVC  DB2_HADR01_P  	--should be primary hadr service port
db2 update db cfg for mysample using HADR_REMOTE_HOST hadr02 	--should be standby ip add or hostname
db2 update db cfg for mysample using HADR_REMOTE_SVC DB2_HADR02_S	--should be standby hadr service port
db2 update db cfg for mysample using HADR_REMOTE_INST db2v10i   --should be standy server's instance
db2 update db cfg for mySAMPLE using HADR_SYNCMODE nearsync	--sychronization mode
```
#### 2.5 Configure database parameters of HADR on standby
```
db2 update db cfg for mysample using HADR_LOCAL_HOST hadr02 	--should be standby ip addr or hostname
db2 update db cfg for mysample using HADR_LOCAL_SVC DB2_HADR02_S	--should be standby hadr service port
db2 update db cfg for mysample using HADR_REMOTE_HOST hadr01	--should be primary ip addr or hostname
db2 update db cfg for mysample using HADR_REMOTE_SVC DB2_HADR01_P	--should be primary hadr service port
db2 update db cfg for mysample using HADR_REMOTE_INST db2v10i
db2 update db cfg for mySAMPLE using HADR_SYNCMODE nearsync
```
Be aware, the value for <code>hadr_local_svc</code> on the primary or standby database systems cannot be the same as the value of svcename or svcename +1 on their corresponding hosts. For instance,my instance SVCENAME is 50000, then you cannot use 50000 or 50001 for the HADR service port.
#### 2.6 Start HADR on primary and standby
Recommend startup standby first.
```
--on standy
db2 start hadr on database mysample as standby
--on primary
db2 start hadr on database mysample as primary
```
If maintenance tasks hanppened, such as OS upgrade, fix pack need to restart the HADR, do not issue any db2 stop hadr on db command at any host.
<li>Stop HADR</li>
```
--First on standby
db2 deactivate db mysample
db2stop
--Then on primary
db2 deactivate database mysample
db2stop
```
<li>Start HADR</li>
```
--First on the primary
db2start
db2 activate db mysample
--Then on standby
db2start
db2 activate db mysample
```
### 3. RoS in HADR
From DB2 version v9.7.1, standby database is supported enable read on standby feature, this feature is enable query statements with UR isolation level in standby while log reply is happening simultaneously.
```
--enable RoS in standby
[db2v10i@hadr02 ~]$ db2set DB2_HADR_ROS=ON
[db2v10i@hadr02 ~]$ db2set DB2_STANDBY_ISO=UR
[db2v10i@hadr02 ~]$ db2 terminate
DB20000I  The TERMINATE command completed successfully.
[db2v10i@hadr02 ~]$ db2 deactivate db mysample
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2v10i@hadr02 ~]$ db2stop && db2start
03/09/2016 18:42:53     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
03/09/2016 18:42:54     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
```
Let's start HADR and have a test.
```
[db2v10i@hadr02 ~]$ db2 start hadr on db mysample as standby
DB20000I  The START HADR ON DATABASE command completed successfully.
```
In primary, we create a new table, and insert some records into the table, see if it can be queried or not in standby.
```
[db2v10i@hadr01 ~]$ db2 "create table t like syscat.tables"
DB20000I  The SQL command completed successfully.
[db2v10i@hadr01 ~]$ db2 "insert into t select * from syscat.tables"
DB20000I  The SQL command completed successfully.
[db2v10i@hadr01 ~]$ db2 "select count(*) from t"
        438
```
Finding the result of standby server.
```
[db2v10i@hadr02 ~]$ db2 connect to mysample
[db2v10i@hadr02 ~]$ db2 "select count(*) from t with ur"
        438
```
There are till some restrictions on RoS, standby database only support UR isolation level, Automatic Client Reroute not supported in RoS, when DDL replaying on standby, user cannot access to the standby in RoS, also with some maintenance tasks such as <code>runstats</code>, <code>reorg</code>, so, it's recommended when doing maintenance tasks, choose a maintenance window to accomplish it.
### 4. Planned takeover
Takeover also call switch roles, when planning some maintenance tasks in primary server, to minimize the downtime, standby can takeover as a primary role. Switching roles only be avaliable when databases are in peer state. And do not forget to rerouter the clients either manually or by ACR after takeover.
```
--monitoring the states of HADR by db2pd or get snapshot command
[db2v10i@hadr01 ~]$ db2 get snapshot for database on mysample |grep -i hadr -A 6
HADR Status
  Role                   = Primary
  State                  = Peer
  Synchronization mode   = Nearsync
  Connection status      = Connected, 03/09/2016 18:44:10.197363
  Heartbeats missed      = 0
  Local host             = hadr01
  Local service          = HADR01
  Remote host            = hadr02
  Remote service         = HADR02
  Remote instance        = db2v10i
  timeout(seconds)       = 120
  Primary log position(file, page, LSN) = S0000004.LOG, 654, 000000000355638A
  Standby log position(file, page, LSN) = S0000004.LOG, 654, 000000000355638A
  Log gap running average(bytes) = 158
```
If the state is in peer, we can takeover now.
```
--on standby
[db2v10i@hadr02 ~]$ db2 takeover hadr on db mysample
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
--finding the roles by using get snapshot command, you will find the roles are changed
[db2v10i@hadr01 ~]$ db2 get snapshot for database on mysample |grep -i hadr -A 6
HADR Status
  Role                   = Standby
  State                  = Peer
  Synchronization mode   = Nearsync
  Connection status      = Connected, 03/09/2016 18:44:10.197363
  Heartbeats missed      = 0
  Local host             = hadr01
  Local service          = HADR01
  Remote host            = hadr02
  Remote service         = HADR02
  Remote instance        = db2v10i
  timeout(seconds)       = 120
  Primary log position(file, page, LSN) = S0000004.LOG, 654, 0000000003556527
  Standby log position(file, page, LSN) = S0000004.LOG, 654, 0000000003556527
  Log gap running average(bytes) = 864220
```
When the maintenance jobs are done, you can switch over the roles.
Besides switch over, there's a takeover called takeover by force, it means failover, if the primary not functional, you should use takeover by force. When failover happened, it means the HADR not functional anymore, you need re-build the HADR by backup image.
### 5. HADR monitoring
You can monitor HADR status by <code>db2pd</code> or db2 get snapshot command.
#### 5.1 db2pd
```
[db2v10i@hadr01 ~]$ db2pd -d mysample -hadr show detail
Database Member 0 -- Database MYSAMPLE -- Active -- Up 0 days 00:00:33 -- Date 03/14/2016 15:48:38
                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = NEARSYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                  PRIMARY_MEMBER_HOST = hadr01
                     PRIMARY_INSTANCE = db2v10i
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = hadr02
                     STANDBY_INSTANCE = db2v10i
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
             HADR_CONNECT_STATUS_TIME = 03/14/2016 15:48:06.525597 (1457941686)
          HEARTBEAT_INTERVAL(seconds) = 30
                HADR_TIMEOUT(seconds) = 120
        TIME_SINCE_LAST_RECV(seconds) = 3
             PEER_WAIT_LIMIT(seconds) = 0
           LOG_HADR_WAIT_CUR(seconds) = 0.000
    LOG_HADR_WAIT_RECENT_AVG(seconds) = 0.000000
   LOG_HADR_WAIT_ACCUMULATED(seconds) = 0.000
                  LOG_HADR_WAIT_COUNT = 0
SOCK_SEND_BUF_REQUESTED,ACTUAL(bytes) = 0, 19800
SOCK_RECV_BUF_REQUESTED,ACTUAL(bytes) = 0, 87380
            PRIMARY_LOG_FILE,PAGE,POS = S0000001.LOG, 0, 44836001
            STANDBY_LOG_FILE,PAGE,POS = S0000001.LOG, 0, 44836001
                  HADR_LOG_GAP(bytes) = 0
     STANDBY_REPLAY_LOG_FILE,PAGE,POS = S0000001.LOG, 0, 44836001
       STANDBY_RECV_REPLAY_GAP(bytes) = 0
                     PRIMARY_LOG_TIME = 03/10/2016 15:55:08.000000 (1457596508)
                     STANDBY_LOG_TIME = 03/10/2016 15:55:08.000000 (1457596508)
              STANDBY_REPLAY_LOG_TIME = 03/10/2016 15:55:08.000000 (1457596508)
         STANDBY_RECV_BUF_SIZE(pages) = 512
             STANDBY_RECV_BUF_PERCENT = 0
           STANDBY_SPOOL_LIMIT(pages) = 0
                 PEER_WINDOW(seconds) = 0
             READS_ON_STANDBY_ENABLED = N
```
#### 5.2 db2 get snapshot
```
[db2v10i@hadr01 ~]$ db2 get snapshot for database on mysample |grep -i hadr -A 6
Catalog network node name                  = hadr01
Operating system running at database server= LINUXAMD64
Location of the database                   = Local
First database connect timestamp           = 03/14/2016 15:48:05.953771
Last reset timestamp                       =
Last backup timestamp                      = 03/10/2016 15:50:28.000000
Snapshot timestamp                         = 03/14/2016 15:56:55.513115
--
HADR Status
  Role                   = Primary
  State                  = Peer
  Synchronization mode   = Nearsync
  Connection status      = Connected, 03/14/2016 15:48:06.525597
  Heartbeats missed      = 0
  Local host             = hadr01
  Local service          = 57000
  Remote host            = hadr02
  Remote service         = 58000
  Remote instance        = db2v10i
  timeout(seconds)       = 120
  Primary log position(file, page, LSN) = S0000001.LOG, 0, 0000000002AC24A1
  Standby log position(file, page, LSN) = S0000001.LOG, 0, 0000000002AC24A1
  Log gap running average(bytes) = 0
```
#### 5.3 by catalog view
```
[db2v10i@hadr01 ~]$ db2 "select substr(DB_NAME,1,10) as DBNAME,substr(HADR_CONNECT_STATUS,1,10) as connect_stat, \
substr(HADR_ROLE,1,10) as ROLE,substr(HADR_STATE,1,10) as state,substr(HADR_SYNCMODE,1,10) as syncmode, \
substr(HADR_HEARTBEAT,1,10) as heartbeat,substr(HADR_LOCAL_HOST,1,10) as local,\
substr(HADR_REMOTE_HOST,1,10) as remote, SNAPSHOT_TIMESTAMP from sysibmadm.snaphadr"

DBNAME     CONNECT_STAT ROLE       STATE      SYNCMODE   HEARTBEAT  LOCAL      REMOTE     SNAPSHOT_TIMESTAMP
---------- ------------ ---------- ---------- ---------- ---------- ---------- ---------- --------------------------
MYSAMPLE   CONNECTED    PRIMARY    PEER       NEARSYNC   0          hadr01     hadr02     2016-03-14-15.58.03.220468
```
### 6. ACR
Automatic Client Reroute is a feature that enables a DB2 client to recover from a loss of connection to the DB2 server by rerouting the connection to an alternate server.This automatic connection rerouting occurs automatically.
#### 6.1 Catalog primary DB on clients
```
[db2cae@db2client ~]$ db2 catalog tcpip node hadr01 remote 192.168.56.101 server 50000
DB20000I  The CATALOG TCPIP NODE command completed successfully.
[db2cae@db2client ~]$ db2 catalog database mysample as mysample at node hadr01
DB20000I  The CATALOG DATABASE command completed successfully.
```
##### 6.2 Configure ACR in primary and standby
```
--on primary
[db2v10i@hadr01 ~]$ db2 update alternate server for database mysample using hostname hadr02 port 50000
DB20000I  The UPDATE ALTERNATE SERVER FOR DATABASE command completed successfully.
--on standby
[db2v10i@hadr02 ~]$ db2 update alternate server for database mysample using hostname hadr01 port 50000
DB20000I  The UPDATE ALTERNATE SERVER FOR DATABASE command completed successfully.
```
#### 6.3 Running query in client while takeover occur
```
[db2cae@db2client ~]$ db2 connect to mysample user db2v10i
[db2cae@db2client ~]$ db2 "select count(*) from t"
1
-----------
        438
  1 record(s) selected.
```
Let the standby takeover as primary, do not disconnect client's connection by manually.
```
[db2v10i@hadr02 ~]$ db2 takeover hadr on db mysample
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
--query continue while takeover occur
[db2cae@db2client ~]$ db2 "select count(*) from t"
SQL30108N  A connection failed in an automatic client reroute environment. Any
associated transaction was rolled back. Host name or IP address: "hadr02".
Service name or port number: "50000". Reason code: "1". Connection failure
code: "2". Underlying error: "*".  SQLSTATE=08506
[db2cae@db2client ~]$ db2 "select count(*) from t"
1
-----------
        438
  1 record(s) selected.
```
You cannot use the automatic client reroute (ACR) if you enable the read on standby feature.

### 7. New feature on HADR examples
#### 7.1 Delayed reply and log spooling
The database configuration parameter <code>HADR_REPLAY_DELAY</code> define the amount time of delayed replay in seconds (this parameter should only be configured in standby), value zero means turn off this feature, which is the default value. Log delayed replay is depend on the following rules, it's important to sychronize the system clock between primary and standby system.
```
--log replay occurred follow below rules
(current time on the standby - value of the hadr_replay_delay configuration parameter) >=
	timestamp of the committed log record
```
It's recommended to configure log spooling feature when enabling delayed replay. For the convience of query on the standby, I also enable the RoS on the standby.
Restrictions on delayed relay:
<li>Only support superasync synchronization mode</li>
<li>Not support take over while enable this feature</li>
#### 7.2 examples for delayed reply and log spooling
Modify sychronization mode, the database configuration parameter <code>HADR_SYNCMODE</code> is not dynamic, every change to the sychronization mode requires a database restart for both primary and standby databases.
```
--change sync mode to super-async
[db2v10i@hadr01 ~]$ db2 update db cfg for mysample using HADR_SYNCMODE  superasync
[db2v10i@hadr02 ~]$ db2 update db cfg for mysample using HADR_SYNCMODE  superasync
--enable log spooling in standby
[db2v10i@hadr02 ~]$ db2 update db cfg using HADR_SPOOL_LIMIT 25600
DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
```
Enable RoS on standby:
```
[db2v10i@hadr02 ~]$ db2set DB2_STANDBY_ISO=UR
[db2v10i@hadr02 ~]$ db2set DB2_HADR_ROS=ON
[db2v10i@hadr02 ~]$ db2set
DB2_STANDBY_ISO=UR
DB2_HADR_ROS=ON
--recycle the instance to enable RoS
[db2v10i@hadr02 ~]$ db2 deactivate db mysample
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2v10i@hadr02 ~]$ db2 stop hadr on db mysample
DB20000I  The STOP HADR ON DATABASE command completed successfully.
[db2v10i@hadr02 ~]$ db2stop && db2start
03/14/2016 16:48:26     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
03/14/2016 16:48:28     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
[db2v10i@hadr02 ~]$ db2 start hadr on db mysample as standby
DB20000I  The START HADR ON DATABASE command completed successfully.
[db2v10i@hadr01 ~]$ db2pd -d mysample -hadr|grep -i read
             READS_ON_STANDBY_ENABLED = Y
```
Enable delayed replay feature:
```
--configure delay replay parameter on standby
[db2v10i@hadr02 ~]$ db2 update db cfg for mysample using HADR_REPLAY_DELAY 600
--restart hadr on both servers
[db2v10i@hadr02 ~]$ db2 deactivate db mysample
[db2v10i@hadr02 ~]$ db2 stop hadr on db mysample
[db2v10i@hadr01 ~]$ db2 stop hadr on db mysample
[db2v10i@hadr02 ~]$ db2 start hadr on db mysample as standby
[db2v10i@hadr01 ~]$ db2 start hadr on db mysample as primary
```
Now the HADR status should be catchup:
```
[db2v10i@hadr01 ~]$ db2pd -d mysample -hadr|grep -i state
                           HADR_STATE = REMOTE_CATCHUP
```
Create a test table and insert one rows in primary:
```
[db2v10i@hadr01 ~]$ db2 "create table t(id int, name char(20))"
DB20000I  The SQL command completed successfully.
[db2v10i@hadr01 ~]$ db2 "insert into t values(51369,'FUNG')"
DB20000I  The SQL command completed successfully.
```
Even if we enabled RoS on standby now, but because of RoS restrictions on DDL, when replaying DDL happened on standby, cannot access to the standby,
```
[db2v10i@hadr02 ~]$ db2 connect to mysample
SQL1776N  The command cannot be issued on an HADR standby database. Reason  code = "4".
[db2v10i@hadr01 ~]$ db2pd -d mysample -hadr |grep -i replay_only
    STANDBY_REPLAY_ONLY_WINDOW_ACTIVE = Y                   #replay only window is active
     STANDBY_REPLAY_ONLY_WINDOW_START = 03/14/2016 16:56:19.000000 (1457945779)
STANDBY_REPLAY_ONLY_WINDOW_TRAN_COUNT = 1
```
Trying to connect in standby 10 mins later:
```
[db2v10i@hadr02 ~]$ db2 connect to mysample
[db2v10i@hadr02 ~]$ date
Mon Mar 14 17:06:27 CST 2016
[db2v10i@hadr02 ~]$ db2 "select * from t"

ID          NAME
----------- --------------------
      51369 FUNG

  1 record(s) selected.
```
Now we insert another rows and delete first rows in primary, and query the table from standby:
```
[db2v10i@hadr01 ~]$ db2 "insert into t values(111,'KONG')"
DB20000I  The SQL command completed successfully.
[db2v10i@hadr01 ~]$ db2 "delete from t where name='FUNG'"
DB20000I  The SQL command completed successfully.
[db2v10i@hadr02 ~]$ db2 "select * from t"
ID          NAME
----------- --------------------
      51369 FUNG
        111 KONG

  2 record(s) selected.
```
What happened? The deleted data still in the standby with inserted data? If delayed replay works, we should only see the first rows in the table. Actually, in the DB2, not all operations in the log recorded the timestamp, the automatic committed operation contains 2 operations: insert and commit, when insert operation is transferred to standby, this insert operation is replayed first,but not committed.So you can query this row by UR isolation. Next, I stop the HADR on the standby, and rollforward it as complete:
```
[db2v10i@hadr02 ~]$ db2 terminate
[db2v10i@hadr02 ~]$ db2 deactivate db mysample
[db2v10i@hadr02 ~]$ db2 stop hadr on db mysample
[db2v10i@hadr02 ~]$ db2 rollforward db mysample complete
                                 Rollforward Status
 Input database alias                   = mysample
 Number of members have returned status = 1
 Member ID                              = 0
 Rollforward status                     = not pending
 Next log file to be read               =
 Log files processed                    = S0000000.LOG - S0000001.LOG
 Last committed transaction             = 2016-03-14-08.58.44.000000 UTC
DB20000I  The ROLLFORWARD command completed successfully.

[db2v10i@hadr02 ~]$ db2 connect to mysample
[db2v10i@hadr02 ~]$ db2 "select * from t"
ID          NAME
----------- --------------------
      51369 FUNG
  1 record(s) selected.
```
The query result is what we expects, insert and delete operations are all delayed replay, so the only data we can see in the table is the original rows. but remember, if you issue rollforward command to the standby database, you should rebuild the HADR via backup image. Never do that in a production environment.
### 8. Conclusions
The mechanism of HADR is primary database sends the log contents to be replayed on standby. There are 2 special euds processes to handle the log transmission,<code>db2hadrp</code> on primary and <code>db2hadrs</code> on standby.
```
[db2v10i@hadr01 ~]$ db2pd -edus|grep -i hadr
30        139899399300864      3405                 db2hadrp (MYSAMPLE) 0                  0.000000     0.020000
[db2v10i@hadr02 ~]$ db2pd -edus |grep -i hadr
30        140435959834368      3299                 db2hadrs (MYSAMPLE) 0                  0.010000     0.040000
```
There are 2 important parameters:
<code>HADR_TIMEOUT</code>: Specified the amount of time (in seconds) to wait before HADR considers the communication lost between database pairs.If an HADR database does not receive any communication from its partner database for longer than the length of time specified by the hadr_timeout database configuration parameter, then the database concludes that the connection with the partner database is lost.
<code>HADR_PEER_WINDOW</code>: Specified the amount of time (in seconds) in which the primary database suspends a transaction after the database pairs have entered disconnect state.

Next topic will be how to rolling update (fix pack) in HADR.
</br>
<b>EOF</b></br>



