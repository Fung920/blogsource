---
layout: post
title: "Install DB2 in Linux"
date: 2015-08-28 14:39:30
comments: false
categories: db2
tags: db2
keywords: db2 
description: How to install db2 server in Linux
---
Install DB2 server in Linux may not as difficult as Oracle database server installation in Linux, it just need a little change to install DB2 server in Linux, espacially in silent mode. This post just a practice of my homework. And any suggestions will be welcome.
<!--more-->
### I. pre-install tasks
My test environment OS is RHEL6.5 X64, and DB2 version is v10.1. Since v9.7.2, the kernal parameters do not need to modify manually, as the database manager modifies kernal parameters dynamically when it starts. And not like Oracle, DB2 with response file installation no need to create account in OS, as you can specify users in the response file.
#### 1. edit /etc/hosts, add hostname and ip address, and disable selinux
```
[root@hadr01 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.56.60   hadr01
192.168.56.61   hadr02

[root@hadr01 worktmp]# sed -i '/^SELINUX/s/enforcing/disabled/g' /etc/selinux/config
```
#### 2. extract DB2 binary, and run prerecheck
```
[root@hadr01 worktmp]# ls -ltr
total 734968
drwxr-xr-x. 3 root root      4096 Apr  4  2012 server
-rw-r--r--. 1 root root       999 Sep  5 10:24 db2ese_c.lic
-rw-r--r--. 1 root root 752588810 Sep  5 10:24 v10.1_linuxx64_server.tar.gz
-rw-r--r--. 1 root root      1528 Sep  5 10:59 db2ese.rsp
[root@hadr01 worktmp]# cd server/
[root@hadr01 server]# ./db2prereqcheck 
```
Fix the warnings or errors until all requirements <code>db2prereqcheck</code> matched.
```
--For TSAMP installation
[root@hadr01 server]# yum install libstdc++.i686 perl ksh pam.i686 -y
```
### 3. create the response file using the sample responses file
The sample response file located at <code>server/db2/platform/samples/</code>, 
```
[root@hadr01 samples]# pwd
/worktmp/server/db2/linuxamd64/samples
[root@hadr01 samples]# ls -ltr
total 368
-r--r--r--. 1 bin bin 42350 Apr  4  2012 db2pe.rsp
-r--r--r--. 1 bin bin 54144 Apr  4  2012 db2ese.rsp
-r--r--r--. 1 bin bin 43397 Apr  4  2012 db2consv.rsp
-r--r--r--. 1 bin bin 27826 Apr  4  2012 db2client.rsp
-r--r--r--. 1 bin bin 54035 Apr  4  2012 db2wse.rsp
-r--r--r--. 1 bin bin 54435 Apr  4  2012 db2aese.rsp
-r--r--r--. 1 bin bin  9440 Apr  4  2012 db2un.rsp
-r--r--r--. 1 bin bin 43713 Apr  4  2012 db2exp.rsp
-r--r--r--. 1 bin bin 26381 Apr  4  2012 db2rtcl.rsp

[root@hadr01 samples]# mv db2ese.rsp /worktmp/
[root@hadr01 samples]# cd /worktmp/
```
Edit the response file to meet your environment.
```
[root@hadr01 worktmp]# cat db2ese.rsp 
LIC_AGREEMENT       = ACCEPT
PROD       = ENTERPRISE_SERVER_EDITION
FILE       = /opt/ibm/db2/V10.1
INSTALL_TYPE       = CUSTOM
COMP       = TSAMP
COMP       = APPLICATION_DEVELOPMENT_TOOLS
COMP       = COMMUNICATION_SUPPORT_TCPIP
COMP       = JDK
COMP       = JAVA_SUPPORT
COMP       = BASE_DB2_ENGINE
COMP       = REPL_CLIENT
COMP       = LDAP_EXPLOITATION
COMP       = INSTANCE_SETUP_SUPPORT
COMP       = DB2_SAMPLE_DATABASE
COMP       = ACS
COMP       = SQL_PROCEDURES
COMP       = DB2_DATA_SOURCE_SUPPORT
COMP       = BASE_CLIENT
COMP       = CONNECT_SUPPORT
* ----------------------------------------------
*  Instance properties           
* ----------------------------------------------
INSTANCE                        = inst1
inst1.TYPE                      = ese
*  Instance-owning user
inst1.NAME                      = db2v10i
inst1.GROUP_NAME                = db2iadm1
inst1.HOME_DIRECTORY            = /home/db2v10i
inst1.PASSWORD                  = db24ever
ENCRYPTED                       = inst1.PASSWORD
inst1.AUTOSTART                 = YES
inst1.SVCENAME                  = db2c_db2v10i
inst1.PORT_NUMBER               = 50000
inst1.FCM_PORT_NUMBER           = 60000
inst1.MAX_LOGICAL_NODES         = 4
inst1.CONFIGURE_TEXT_SEARCH     = NO
*  Fenced user
inst1.FENCED_USERNAME           = db2fenc1
inst1.FENCED_GROUP_NAME         = db2fadm1
inst1.FENCED_HOME_DIRECTORY     = /home/db2fenc1
inst1.FENCED_PASSWORD           = db24ever
ENCRYPTED                       = inst1.FENCED_PASSWORD
*-----------------------------------------------
*  Installed Languages 
*-----------------------------------------------
LANG                            = EN
```
### II. Install DB2 using respons file
```
[root@hadr01 install]# cd /worktmp/server/
[root@hadr01 server]# ./db2setup -r /worktmp/db2ese.rsp -l /worktmp/db2setup.log -t /worktmp/db2setup.trc
DBI1191I  db2setup is installing and configuring DB2 according to the
      response file provided. Please wait.


The execution completed successfully.

For more information see the DB2 installation log at "/worktmp/db2setup.log".
```
### III. Post-install tasks
#### 1. Configure firewall for DB2 service port
You dont need to manual configure database manager parameter "scvename", as the response file already did. What we need to do is opening the service port in firewall.
```
[root@hadr01 ~]# cat /etc/sysconfig/iptables
# Firewall configuration written by system-config-firewall
# Manual customization of this file is not recommended.
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
#For DB2 services
#-A INPUT -p tcp --dport 50000 -j ACCEPT
#-A OUTPUT -p tcp --sport 50000 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 50000 -j ACCEPT
-A OUTPUT -m state --state NEW -m tcp -p tcp --sport 50000 -j ACCEPT

-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT

[root@hadr01 server]# iptables -L -n |grep 50000
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:50000 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp spt:50000 
[root@hadr01 ~]# cat /etc/services |grep 50000
db2c_db2v10i    50000/tcp
[db2v10i@hadr01 ~]$ db2 get dbm cfg |grep -i svcename
 TCP/IP Service name                          (SVCENAME) = db2c_db2v10i
```
#### 2. Add license
```
[db2v10i@hadr01 ~]$ db2licm -a /worktmp/db2ese_c.lic

LIC1402I  License added successfully.

LIC1426I  This product is now licensed for use as outlined in your License Agreement.  
USE OF THE PRODUCT CONSTITUTES ACCEPTANCE OF THE TERMS OF THE IBM LICENSE AGREEMENT, 
LOCATED IN THE FOLLOWING DIRECTORY: "/opt/ibm/db2/V10.1/license/en_US.iso88591"
[db2v10i@hadr01 ~]$ db2licm -l
Product name:                     "DB2 Enterprise Server Edition"
License type:                     "CPU Option"
Expiry date:                      "Permanent"
Product identifier:               "db2ese"
Version information:              "10.1"
Enforcement policy:               "Soft Stop"
```
#### 3. Create database
--Create sample database (optional)
```
[root@hadr01 ~]# mkdir -p /db2/{data,backup,arch,log}
[root@hadr01 ~]# chown -R db2v10i:db2iadm1 /db2/
[db2v10i@hadr01 ~]$ db2sampl -dbpath /db2/data -name mysample -sql -force -verbose
  Creating database "mysample" on path "/db2/data"...
  Connecting to database "mysample"...
  Creating tables and data in schema "DB2V10I"...
  'db2sampl' processing complete.
```
--Create user defined database
```
[db2v10i@hadr01 ~]$ db2 create db testdb automatic storage yes on '/db2/data' DBPATH on '/db2/data' using codeset UTF-8 TERRITORY US
DB20000I  The CREATE DATABASE command completed successfully.
```
#### 4. Configure client connections
On the clients:   
First, catalog node on clients:
```
[db2inst1@db2v10 ~]$ db2 catalog tcpip node hadr01 remote 192.168.56.60 server 50000
DB20000I  The CATALOG TCPIP NODE command completed successfully.
DB21056W  Directory changes may not be effective until the directory cache is 
refreshed.
[db2inst1@db2v10 ~]$ db2 list node directory
 Node Directory
 Number of entries in the directory = 1
Node 1 entry:
 Node name                      = HADR01
 Comment                        =
 Directory entry type           = LOCAL
 Protocol                       = TCPIP
 Hostname                       = 192.168.56.60
 Service name                   = 50000
```
Second, catalog database on clients:
```
[db2inst1@db2v10 ~]$ db2 catalog database testdb as testdb at node hadr01
DB20000I  The CATALOG DATABASE command completed successfully.
DB21056W  Directory changes may not be effective until the directory cache is 
refreshed.
[db2inst1@db2v10 ~]$ db2 list db directory
 System Database Directory
 Number of entries in the directory = 1
Database 1 entry:
 Database alias                       = TESTDB
 Database name                        = TESTDB
 Node name                            = HADR01
 Database release level               = 10.00
 Comment                              =
 Directory entry type                 = Remote
 Catalog database partition number    = -1
 Alternate server hostname            =
 Alternate server port number         =
```
In the output, where "Directory entry type" equals "remote", it means from the database is on a remote server, oppositely, where the "Directory entry type" equals "indirect", it means a local database.   
Now, we can connect to the server in the client:
```
[db2inst1@db2v10 ~]$ db2 connect to testdb user db2v10i
Enter current password for db2v10i: 
SQL30081N  A communication error has been detected. Communication protocol 
being used: "TCP/IP".  Communication API being used: "SOCKETS".  Location 
where the error was detected: "192.168.56.60".  Communication function 
detecting the error: "connect".  Protocol specific error code(s): "113", "*", 
"*".  SQLSTATE=08001
```
Something wrong with the client connection. And I found that the issue was firewall configure not correct. In RHEL6, in firewall configuration file <code>/etc/sysconfig/iptables</code>, if you need to open the other port, your configure command need to below 22 port. For example:
```
[root@hadr01 ~]# cat /etc/sysconfig/iptables
# Firewall configuration written by system-config-firewall
# Manual customization of this file is not recommended.
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
#For DB2 services, open other port need to follow the 22 port
#-A INPUT -p tcp --dport 50000 -j ACCEPT
#-A OUTPUT -p tcp --sport 50000 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 50000 -j ACCEPT
-A OUTPUT -m state --state NEW -m tcp -p tcp --sport 50000 -j ACCEPT

-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```
Try to reconnect:
```
[db2inst1@db2v10 ~]$ db2 connect to testdb user db2v10i using db24ever
   Database Connection Information
 Database server        = DB2/LINUXX8664 10.1.0
 SQL authorization ID   = DB2V10I
 Local database alias   = TESTDB
```
#### 5.Configure archive log using disk
DB2 offer 2 methods to manager transaction logs, circular and archive, most of production environment using archive log mode.
```
[db2v10i@hadr01 ~]$ mkdir -p /db2/log/db2v10i/mysample
[db2v10i@hadr01 ~]$ db2 update db cfg for mysample using NEWLOGPATH /db2/log/db2v10i/mysample
DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
SQL1363W  Database must be deactivated and reactivated before the changes to 
one or more of the configuration parameters will be effective.
[db2v10i@hadr01 ~]$ db2 terminate
DB20000I  The TERMINATE command completed successfully.
[db2v10i@hadr01 ~]$ db2 deactivate database mysample
DB20000I  The DEACTIVATE DATABASE command completed successfully.
[db2v10i@hadr01 ~]$ db2 activate database mysample
DB20000I  The ACTIVATE DATABASE command completed successfully.
[db2v10i@hadr01 ~]$ db2 get db cfg for mysample |grep -i log
 Log retain for recovery status                          = NO
 User exit for logging status                            = NO
 Catalog cache size (4KB)              (CATALOGCACHE_SZ) = (MAXAPPLS*5)
 Log buffer size (4KB)                        (LOGBUFSZ) = 256
 Log file size (4KB)                         (LOGFILSIZ) = 1000
 Number of primary log files                (LOGPRIMARY) = 3
 Number of secondary log files               (LOGSECOND) = 10
 Changed path to log files                  (NEWLOGPATH) = 
 Path to log files                                       = /db2/log/db2v10i/mysample/NODE0000/LOGSTREAM0000/
```
Change default circular log mode to archive log
```
[db2v10i@hadr01 ~]$ db2 get db cfg for mysample |grep -i LOGARCHMETH1
 First log archive method                 (LOGARCHMETH1) = OFF
 Archive compression for logarchmeth1    (LOGARCHCOMPR1) = OFF
 Options for logarchmeth1                  (LOGARCHOPT1) = 

[db2v10i@hadr01 ~]$ db2 update db cfg for mysample using LOGARCHMETH1 disk:db2/arch/mysample
DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
SQL1363W  Database must be deactivated and reactivated before the changes to one or more of the configuration parameters will be effective.
[db2v10i@hadr01 ~]$ db2 terminate
DB20000I  The TERMINATE command completed successfully.
[db2v10i@hadr01 ~]$ db2 deactivate database mysample
DB20000I  The DEACTIVATE DATABASE command completed successfully.
```
After modifying the LOGARCHMETH1 parameter the database will need  an offline full backup:
```
[db2v10i@hadr01 ~]$ db2 backup db mysample to /db2/backup/mysample/
Backup successful. The timestamp for this backup image is : 20150906162210

[db2v10i@hadr01 ~]$ db2 activate database mysample
DB20000I  The ACTIVATE DATABASE command completed successfully.
[db2inst1@hadr01 ~]$ db2 connect to testdb
[db2v10i@hadr01 ~]$ db2 get db cfg for mysample |grep -i LOGARCHMETH1
 First log archive method                 (LOGARCHMETH1) = DISK:/db2/arch/mysample/
```
Manual archive a log, and check the archive whether works or not:
```
[db2v10i@hadr01 ~]$ ll /db2/arch/mysample/db2v10i/MYSAMPLE/NODE0000/LOGSTREAM0000/C0000000/
total 0

[db2v10i@hadr01 ~]$ db2 connect reset
DB20000I  The SQL command completed successfully.
[db2v10i@hadr01 ~]$ db2 archive log for database mysample
DB20000I  The ARCHIVE LOG command completed successfully.

[db2v10i@hadr01 ~]$ ll /db2/arch/mysample/db2v10i/MYSAMPLE/NODE0000/LOGSTREAM0000/C0000000/
total 12
-rw-r----- 1 db2v10i db2iadm1 12288 Sep  6 16:25 S0000000.LOG
```
#### 6.Configure incremental tracking mode
If you want to backup the database in the incremental or delta mode, the trackmod configuration parameter must be set to "ON"
```
[db2v10i@hadr01 ~]$ db2 update db cfg for mysample using trackmod on
DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
SQL1363W  Database must be deactivated and reactivated before the changes to one or more of the configuration parameters will be effective.
```
After changing this parameter to "ON" you will also need to take a full backup (online or offline) to establish the baseline for incremental or delta backups. 
```
[db2v10i@hadr01 ~]$ db2 backup database mysample online to /db2/backup/mysample/ include logs
Backup successful. The timestamp for this backup image is : 20150906163124
```

</br>
<b>EOF</b></br>
