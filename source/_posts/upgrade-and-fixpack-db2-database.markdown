---
layout: post
title: "Upgrade and fix pack db2 database"
date: 2016-01-26 21:06:44
comments: false
categories: db2
tags: db2
keywords: upgrade db2 server 
description: how to upgrade and fix pack in db2 server
---
Upgrading to DB2 Version 10.1 is supported from DB2 Version 9.5, DB2 Version 9.7 and DB2 Version 9.8. If you have an earlier version of DB2, you must upgrade to DB2 Version 9.5 before upgrading to DB2 Version 10.1.
<!--more-->
Upgrade DB2 server there are two methods: one is installing new product binary code to a new path, another is installing new product to the old path,it's recommended to install to a new path.
### 1. pre-upgrade tasks
First of all, we need to check the disk space requirement, recommended space list in the below:   
On Linux/UNIX,   
```
/home	> 2GB
/tmp	> 5GB
/var    > 1GB
/install_dir >4G
```
#### 1.1 upgrade check
Before upgrading the database, you can run <code>db2ckupgrade</code> to check whether the database is ready for upgrade or not, commonly, the <code>db2ckupgrade</code> is located in "DIRIMG/db2/OS/utilities/db2ckupgrade/bin" directory.   
Log on as instance owner, issue <code>db2ckupgrade</code> command, as follows: 
```
[db2v97i@db2srv ~]$ db2start
[db2v97i@db2srv ~]$ /worktmp/server/db2/linuxamd64/utilities/db2ckupgrade/bin/db2ckupgrade mysample -l ~/db2ckupgrade.log 
DBT5508I  The db2ckupgrade utility completed successfully. The database or databases can be upgraded.
```
The mysample is the database name and the db2ckupgrade.log that includes details on errors and warnings.
#### 1.2 backup database before upgrading
Performing a full offline backup is recommended, check how many databases are under the instance.
```
[db2v97i@db2srv ~]$ db2 list db directory
[db2v97i@db2srv ~]$ db2 list application
[db2v97i@db2srv ~]$ db2 force application all
[db2v97i@db2srv ~]$ db2 backup db mysample to /db2/backup/db2v97i/mysample/
[db2v97i@db2srv ~]$ db2ckbkp backup_image 
```
#### 1.3 backup database and dbm configurations
<code>db2support</code> collects the information about the database system catalog, database and database manager configuration parameters, DB2 registry variable settings, diagnostic information etc.   
```
[db2v97i@db2srv ~]$ db2support ./ -d mysample -cl 0
```
Or you can backup  manually:
```
--backing up database configurations
[db2v97i@db2srv ~]$ db2 connect to mysample
[db2v97i@db2srv ~]$ db2 GET DB CFG FOR mysample show detail > mysample.cfg
--backing up database manager configurations
[db2v97i@db2srv ~]$ db2 get dbm cfg show detail >dbm.cfg
--backing up DB2 registry variable settings
[db2v97i@db2srv ~]$ db2set -all > reg_db2v97i.cfg
--backing up DDL information
[db2v97i@db2srv ~]$ db2look -d mysample -e -o mysample_tbs.db2 -l -x
--backing up environment variables
[db2v97i@db2srv ~]$ set |grep -i db2 > env_db2v97i.txt
```
Backing up the audit configuration (instance level) by the db2audit command if audit is configured:
```
[db2v97i@db2srv ~]$ db2audit describe > audit_instance-name.cfg
```
### 2. upgrading a DB2 server
#### 2.1 install DB2 v10.1 production
Log on as root user, install DB2 v10.1 production to a different location (as I have done):
```
[db2v97i@db2srv ~]$ db2ls
Install Path  Level   Fix Pack   Special Install Number Install Date           Installer UID 
--------------------------------------------------------------------------------
/opt/ibm/db2/V9.7_FP10   9.7.0.10   10   Tue Sep 29 17:07:52 2015 CST      0 
/opt/ibm/db2/V10.1       10.1.0.0   0    Wed Sep 30 17:07:00 2015 CST      0
```
#### 2.2 upgrade the DB2 instance
Log on as instance owner, stop the instance:
```
[db2v97i@db2srv ~]$ db2 force applications all
[db2v97i@db2srv ~]$ db2stop force
[db2v97i@db2srv ~]$ db2 terminate
```
Log on as root, run <code>db2iupgrade</code> command from the target DB2 version 10.1 code location, such as:
```
[root@db2srv ~]# /opt/ibm/db2/V10.1/instance/db2iupgrade -k db2v97i 
--$DB2DIR/instance/db2iupgrade
[root@db2srv ~]# su - db2v97i
[db2v97i@db2srv ~]$ db2start
10/09/2015 15:45:19     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
[db2v97i@db2srv ~]$ db2level
DB21085I  Instance "db2v97i" uses "64" bits and DB2 code release "SQL10010" with level identifier "0201010E".
Informational tokens are "DB2 v10.1.0.0", "s120403", "LINUXAMD64101", and FixPack "0".
Product is installed at "/opt/ibm/db2/V10.1".
```
#### 2.3 upgrading the DB2 database
Log on as instance owner and using the <code>upgrade database</code> command:
```
[db2v97i@db2srv ~]$ db2 upgrade database mysample
[db2v97i@db2srv ~]$ db2 connect to mysample

   Database Connection Information

 Database server        = DB2/LINUXX8664 10.1.0
 SQL authorization ID   = DB2V97I
 Local database alias   = MYSAMPLE
```
### 3. post-upgrade tasks
#### 3.1 rebind packages
After upgrading instance and database to new version, all application packages written in C, C++, COBOL, FORTRAN, and REXX are marked as invalid, and will be implicitly rebound the first time an application uses them after upgrading database. To eliminate this overhead, you can rebind invalid packages with REBIND or db2rbind command after the upgrade. 
```
[db2v97i@db2srv ~]$ db2rbind mysample -l ./rbind.log all
 Rebind done successfully for database 'MYSAMPLE'.
```
It's recommended to bind the following packages while using SQL/MQ replication:
```
# db2 connect to <db_name>
--Binding SQL Capture packages
[db2v97i@db2srv ~]$ db2 bind /opt/ibm/db2/V10.1/bnd/@capture.lst isolation ur blocking all
--Binding SQL Apply packages
[db2v97i@db2srv ~]$ db2 bind /opt/ibm/db2/V10.1/bnd/@applycs.lst isolation cs blocking all grant public
[db2v97i@db2srv ~]$ db2 bind /opt/ibm/db2/V10.1/bnd/@applyur.lst isolation ur blocking all grant public
--Binding Q Capture packages
[db2v97i@db2srv ~]$ db2 bind /opt/ibm/db2/V10.1/bnd/@qcapture.lst isolation ur blocking all
--Binding Q Apply packages
[db2v97i@db2srv ~]$ db2 bind /opt/ibm/db2/V10.1/bnd/@qapply.lst isolation ur blocking all grant public
# db2 terminate
```
#### 3.2 apply DB2 license
```
[db2v97i@db2srv ~]$ db2licm -a /worktmp/db2ese_c.lic
```
### 4. Install a fix pack
Check the requirements of installation, disk spaces, backup database and configurations, etc..
#### 4.1 stop DB2 process
```
[db2v97i@db2srv ~]$ db2 force applications all
[db2v97i@db2srv ~]$ db2 terminate
[db2v97i@db2srv ~]$ db2stop 
[db2v97i@db2srv ~]$ db2licd -end
```
Prevent the instances from auto-starting:
```
[db2v97i@db2srv ~]$ /opt/ibm/db2/V10.1/instance/db2iauto -off db2v97i
```
#### 4.2 install fix pack
There are 2 methods to install fix pack, using installFixPack to install fix pack to existing location, or install new DB2 product.      
Go to the fix pack image location, as a root, execute installFixPack command to an existing place:
```
./installFixPack -b DB2DIR
```
I'd like to install to a new place. The installation steps are omitted, my new fixpack directory is <code>/opt/ibm/db2/V10.1_FP5</code>.
#### 4.3 post-installation tasks for fix packs
First, run <code>djxlink</code> command.
Upgrade each instance using <cdoe>db2iupdt</code> command by root user.
```
root@db2srv worktmp]# /opt/ibm/db2/V10.1_FP5/instance/db2iupdt db2v97i
[db2v97i@db2srv ~]$ db2level
DB21085I  This instance or install (instance name, where applicable: "db2v97i") 
uses "64" bits and DB2 code release "SQL10015" with level identifier "0206010E".
Informational tokens are "DB2 v10.1.0.5", "s150624", "IP23772", and Fix Pack "5".
Product is installed at "/opt/ibm/db2/V10.1_FP5".
```
Update the catalog by using <code>db2updv10</code> command as instance owner:
```
[root@db2srv worktmp]# find /opt/ibm/db2/V10.1_FP5/ -name "db2updv*"
/opt/ibm/db2/V10.1_FP5/bnd/db2updv10.bnd
/opt/ibm/db2/V10.1_FP5/bin/db2updv10
[db2v97i@db2srv ~]$ /opt/ibm/db2/V10.1_FP5/bin/db2updv10 -d mysample
[db2v97i@db2srv ~]$ db2start
[db2v97i@db2srv ~]$ db2 connect to mysample
--enable autostart
[db2v97i@db2srv ~]$ /opt/ibm/db2/V10.1/instance/db2iauto -on db2v97i
```
Final step, binding the packages:
```
db2 terminate
db2 CONNECT TO mysample
db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/db2schema.bnd BLOCKING ALL GRANT PUBLIC SQLERROR CONTINUE 
db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/@db2ubind.lst BLOCKING ALL GRANT PUBLIC ACTION ADD 
db2 BIND /opt/ibm/db2/V10.1_FP5/bnd/@db2cli.lst BLOCKING ALL GRANT PUBLIC ACTION ADD 
db2 terminate
db2rbind mysample -l ./rbind.log all
[db2v97i@db2srv ~]$ db2level
DB21085I  This instance or install (instance name, where applicable: "db2v97i") 
uses "64" bits and DB2 code release "SQL10015" with level identifier "0206010E".
Informational tokens are "DB2 v10.1.0.5", "s150624", "IP23772", and Fix Pack "5".
Product is installed at "/opt/ibm/db2/V10.1_FP5".

[db2v97i@db2srv ~]$ db2 connect to mysample

   Database Connection Information

 Database server        = DB2/LINUXX8664 10.1.5
 SQL authorization ID   = DB2V97I
 Local database alias   = MYSAMPLE
```

<b>EOF</b></br>   



