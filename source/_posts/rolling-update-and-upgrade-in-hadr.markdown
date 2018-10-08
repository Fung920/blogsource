---
layout: post
title: "Rolling Update and Upgrade in HADR"
date: 2016-03-17 15:20:49
comments: false
categories: db2
tags: hadr
keywords: rolling update, upgrade
description: how to rolling update and upgrade in HADR
---
HADR can give you high availablity while applying fix pack via rolling update. The database or application downtime can be minimize or zero downtime. With properly ACR setting, it means no data loss and it's no visiable downtime to clients.
<!--more-->
Here's an examples how to apply fix pack and upgrade major version in HADR environment. Below is the example's environment. 
```
HOSTNAME	INSTANCE	DBNAME		DBVERSION	ROLE
node1		db2inst1	mysample	10.1.0.0	primary
node2		db2inst1	mysample 	10.1.0.0	standby
```
### 1. Rolling update in HADR with mininal downtime 
The general steps can be done as following:

- 1.Check the HADR status
- 2.Apply fix pack in standby 
- 3.Swithover roles 
- 4.Apply fix pack in original primary
- 5.Swithover HADR roles back to the original primary
- 6.Post-installation tasks in primary

#### 1.1 Install fix pack in standby/primary server
It's a simple task, I install the fix pack in a new directory: /opt/ibm/db2/V10.1_FP5, the original code directory is: /opt/ibm/db2/V10.1.

```
[root@node2 worktmp]# /opt/ibm/db2/V10.1_FP5/install/db2ls

Install Path                       Level   Fix Pack   Special Install Number   Install Date                  Installer UID 
---------------------------------------------------------------------------------------------------------------------
/opt/ibm/db2/V10.1               10.1.0.0        0                            Thu Mar 17 14:30:55 2016 CST             0 
/opt/ibm/db2/V10.1_FP5           10.1.0.5        5                            Thu Mar 17 16:24:42 2016 CST             0 
```
#### 1.2 Take over HADR in standby
Check the HADR status before taking over in standby, rolling update only can be supported with peer state take over. It also measn when sychronization mode is async, we cannot rolling update without any downtime or data loss. 
```
[db2inst1@node2 ~]$ db2pd -d mysample -hadr |grep -i state -A 5
                           HADR_STATE = PEER
                  PRIMARY_MEMBER_HOST = node1
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = node2
                     STANDBY_INSTANCE = db2inst1
```
Deactivate the standby database first, if necessary, stop the standby instance.
```
[db2inst1@node2 ~]$ db2 deactivate db mysample 
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2inst1@node2 ~]$ db2stop
03/17/2016 16:42:19     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
```
Perform instance update command in standby by root user and add license by instance user.
```
[root@node2 worktmp]# /opt/ibm/db2/V10.1_FP5/instance/db2iupdt db2inst1
[root@node2 worktmp]# su - db2inst1
[db2inst1@node2 ~]$ db2licm -a /worktmp/db2ese_c.lic
[db2inst1@node2 ~]$ db2licm -l
Product name:                     "DB2 Enterprise Server Edition"
License type:                     "CPU Option"
Expiry date:                      "Permanent"
Product identifier:               "db2ese"
Version information:              "10.1"
Enforcement policy:               "Soft Stop"
```
Start instance in standby server, activate the database and perform switch roles in standby server.
```
[db2inst1@node2 ~]$ db2start
03/17/2016 16:48:18     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
[db2inst1@node2 ~]$ db2 activate db mysample
DB20000I  The ACTIVATE DATABASE command completed successfully.
[db2inst1@node2 ~]$ db2 takeover hadr on db mysample 
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
```
#### 1.3 Repeat above steps in original primary
```
[db2inst1@node1 ~]$ db2 terminate
[db2inst1@node1 ~]$ db2 deactivate db mysample
[db2inst1@node1 ~]$ db2stop
[root@node1 worktmp]# /opt/ibm/db2/V10.1_FP5/instance/db2iupdt db2inst1
[db2inst1@node1 ~]$ db2licm -a /worktmp/db2ese_c.lic
[db2inst1@node1 ~]$ db2start
[db2inst1@node1 ~]$ db2 activate db mysample
[db2inst1@node1 ~]$ db2pd -d mysample -hadr |grep -i state -A 5
                           HADR_STATE = PEER
                  PRIMARY_MEMBER_HOST = node2
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = node1
                     STANDBY_INSTANCE = db2inst1
```
#### 1.4 Perform take over from original primary database
```
[db2inst1@node1 ~]$ db2 takeover hadr on db mysample 
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
```
#### 1.5 Post-installation tasks in primary database
First, update the database by using <code>db2updv10</code> as instance owner.
```
[db2inst1@node1 ~]$ /opt/ibm/db2/V10.1_FP5/bin/db2updv10 -d mysample
```
Next, bind the packages and rbind.
```
[db2inst1@node1 ~]$ db2 connect to mysample 
[db2inst1@node1 ~]$ db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/db2schema.bnd BLOCKING ALL GRANT PUBLIC SQLERROR CONTINUE
[db2inst1@node1 ~]$ db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/@db2ubind.lst BLOCKING ALL GRANT PUBLIC ACTION ADD
[db2inst1@node1 ~]$ db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/@db2cli.lst BLOCKING ALL GRANT PUBLIC ACTION ADD
[db2inst1@node1 ~]$ db2rbind mysample -l ./rbind.log all
 Rebind done successfully for database 'MYSAMPLE'.
```
You can confirm your update result by using following commands.
```
[db2inst1@node1 ~]$ db2level
[db2inst1@node1 ~]$ db2 "SET SERVEROUTPUT ON"
DB20000I  The SET SERVEROUTPUT command completed successfully.
[db2inst1@node1 ~]$ db2 " BEGIN
>   DECLARE v_version VARCHAR(80);
>   DECLARE v_compat VARCHAR(80);
>   CALL DBMS_UTILITY.DB_VERSION(v_version, v_compat);
>   CALL DBMS_OUTPUT.PUT_LINE('Version: ' || v_version);
>   CALL DBMS_OUTPUT.PUT_LINE('Compatibility: ' || v_compat);
> end"
DB20000I  The SQL command completed successfully.

Version: DB2 v10.1.0.5
Compatibility: DB2 v10.1.0.5

[db2inst1@node1 ~]$ db2pd -v
Instance db2inst1 uses 64 bits and DB2 code release SQL10015 with level identifier 0206010E
Informational tokens are DB2 v10.1.0.5, s150624, IP23772, Fix Pack 5.
```
With these rolling update steps, you can minimize your downtime.    
### 2. Upgrade HADR database across major version
Because rolling upgrade across major version is not supported, when you meet this request, you need to plan database outage, this request is actually requiring a new build HADR, previous post can be referred [HADR in DB2](/hadr-in-db2.html).

The general steps of upgrade major version in single standby database as follow. 

- Break HADR relationship by rolling forward the standby 
  - Deactivate the standby, stop the HADR on standby
  - Issue rolling forward to complete command in standby

```
[db2inst1@node2 ~]$ db2 deactivate db mysample 
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2inst1@node2 ~]$ db2 stop hadr on db mysample 
DB20000I  The STOP HADR ON DATABASE command completed successfully.
[db2inst1@node2 ~]$ db2 rollforward db mysample complete
```

- Install New Version DB2 code on standby and primary 
- Upgrade the instance and apply license on both servers

```
[db2inst1@node1 ~]$ db2 deactivate db sample 
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2inst1@node1 ~]$ db2 stop hadr on db sample 
DB20000I  The STOP HADR ON DATABASE command completed successfully.
[db2inst1@node1 ~]$ db2stop 
03/18/2016 16:41:08     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
[root@node1 worktmp]# /opt/ibm/db2/V10.5/instance/db2iupgrade -k db2inst1
[db2inst1@node1 ~]$ db2licm -a /worktmp/db2ese_c_v10.5.lic 
```

- Upgrade the database on primary

```
[db2inst1@node1 ~]$ db2start
[db2inst1@node1 ~]$ db2 upgrade db mysample
```

- Re-establish HADR 

You can follow the instuctions by previous post [HADR in DB2](/hadr-in-db2.html).

</br>
<b>EOF</b>


