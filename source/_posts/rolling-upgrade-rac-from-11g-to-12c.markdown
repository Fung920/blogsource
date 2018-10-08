---
layout: post
title: "Rolling Upgrade RAC from 11g to 12c"
date: 2016-04-21 14:32:22
comments: false
categories: oracle
tags: upgrade, 12c
keywords: rolling upgrade grid infrastructure
description: Rolling upgrade RAC from 11g to 12c
name: Rolling Upgrade RAC from 11g to 12c
author: Fung Kong
datePublished: April 21, 2016
---
What I mean rolling is Grid Infrastructure upgrade by rolling, but RDBMS still need outage. To minimize the downtime, and I think upgrade with rolling is more easier than non-rolling, so use rolling upgrade GI is advisable. At the end of this post, I will try to reverse the whole process, that is downgrade RDBMS and GI to before version. 
<!--more-->
I already have a two-node 11gr2 RAC installed in my VM machine, and I'll perform a out-of-place upgrade for GI and RDBMS. Out-of-place upgrade can let you rollback or downgrade more easier.    

New directories have been created as below. 

``` bash [Below infomation should be the same on both nodes]
[root@node1 ~]# mkdir -p /u02 
[root@node1 ~]# chown -R grid:oinstall /u02
[grid@node1:/home/grid]$ cp .bash_profile .bash_profile_11g 
[grid@node1:/home/grid]$ sed -i 's/u01/u02/g' .bash_profile
[grid@node1:/home/grid]$ echo $ORACLE_BASE
/u02/app/12.1.0
[grid@node1:/home/grid]$ echo $ORACLE_HOME
/u02/app/grid/12.1.0
[grid@node1:/home/grid]$ mkdir -p $ORACLE_BASE $ORACLE_HOME
[grid@node1:/home/grid]$ chmod -R 775 /u02 
[oracle@node1:/home/oracle]$ cp .bash_profile .bash_profile_11g
[oracle@node1:/home/oracle]$ echo $ORACLE_HOME
/u02/app/oracle/product/12.1.0/db1
[oracle@node1:/home/oracle]$ echo $ORACLE_BASE
/u02/app/oracle
[oracle@node1:/home/oracle]$ mkdir -p $ORACLE_HOME
```

Whenever upgrade a system, remember that upgrade is a high risk operation, backup is always essential, backup the whole database, the GI and RDBMS home, the configuration files, and so on.  Below backup actions is recommended. 

- GI and RDBMS home

- A manually physical OCR/OLR backup

- Full database backup 

### 1. Some restrictions of rolling upgrade GI 

- Not support for block devices or RAW devices 

If the OCR and voting disks is located on raw or block devices, need to migrate them to ASM before upgrade. 

Below table is the compatibility matrix of clusterware which can be upgrade to 12c directly.

<center>Table 1-1 </center>

|Oracle version| Compatibility| 
|----|----|
|Oracle 10.1.0.5 |Direct upgrade possible|
|Oracle 10.2.0.3| Direct upgrade possible|
|Oracle 11.1.0.6| Direct upgrade possible|
|oracle 11.2.0.2| Direct upgrade possible: patch set 11.2.0.2.3 (PSU 3) or later must be applied|


### 2. Pre-upgrade tasks 
Verify node readiness by running pre-upgrade check script under the source code directory:

``` bash
[grid@node1:/worktmp/grid]$ ./runcluvfy.sh stage -pre crsinst -upgrade -rolling \
-src_crshome /u01/app/11gr2/grid -dest_crshome /u02/app/12.1.0/grid \
-dest_version 12.1.0.2.0 -fixup -verbose 
```

Fix violations until the verify script pass. 

### 3. Performing rolling upgrade GI by response file
Before perform upgrade, the GI environment parameters need to unset or set to new one, I have copied a old profile as .bash_profile_11g, and create a new profile .bash_profile, changed the new parameters direct to 12c. 


#### 3.1 Configure the upgrade GI response file 

``` bash
[grid@node1:/home/grid]$ cat gi.rsp 
oracle.install.responseFileVersion=/oracle/install/rspfmt_crsinstall_response_schema_v12.1.0
ORACLE_HOSTNAME=node1.
INVENTORY_LOCATION=/u02/app/oraInventory
SELECTED_LANGUAGES=en
oracle.install.option=UPGRADE
ORACLE_BASE=/u02/app/12.1.0
ORACLE_HOME=/u02/app/grid/12.1.0
oracle.install.asm.OSDBA=asmdba
oracle.install.asm.OSOPER=
oracle.install.asm.OSASM=asmadmin
oracle.install.crs.config.gpnp.scanName=
oracle.install.crs.config.gpnp.scanPort=  
oracle.install.crs.config.ClusterType=STANDARD
oracle.install.crs.config.clusterName=racdb
oracle.install.crs.config.gpnp.configureGNS=false
oracle.install.crs.config.autoConfigureClusterNodeVIP=true
oracle.install.crs.config.gpnp.gnsOption=CREATE_NEW_GNS
oracle.install.crs.config.gpnp.gnsClientDataFile=
oracle.install.crs.config.gpnp.gnsSubDomain=
oracle.install.crs.config.gpnp.gnsVIPAddress=
oracle.install.crs.config.clusterNodes=node1:,node2:
oracle.install.crs.config.networkInterfaceList=
oracle.install.crs.config.storageOption=LOCAL_ASM_STORAGE
oracle.install.crs.config.sharedFileSystemStorage.votingDiskLocations=
oracle.install.crs.config.sharedFileSystemStorage.votingDiskRedundancy=NORMAL
oracle.install.crs.config.sharedFileSystemStorage.ocrLocations=
oracle.install.crs.config.sharedFileSystemStorage.ocrRedundancy=NORMAL               	
oracle.install.crs.config.useIPMI=false
oracle.install.crs.config.ipmi.bmcUsername=
oracle.install.crs.config.ipmi.bmcPassword=
oracle.install.asm.SYSASMPassword=
oracle.install.asm.diskGroup.name=CRS
oracle.install.asm.diskGroup.redundancy=
oracle.install.asm.diskGroup.AUSize=1
oracle.install.asm.diskGroup.disks=
oracle.install.asm.diskGroup.diskDiscoveryString=
oracle.install.asm.monitorPassword=
oracle.install.asm.ClientDataFile=
oracle.install.crs.config.ignoreDownNodes=false		#if force upgrade enable, turn this to true
oracle.install.config.managementOption=NONE
oracle.install.config.omsHost=
oracle.install.config.omsPort=0
oracle.install.config.emAdminUser=
oracle.install.config.emAdminPassword=
```

#### 3.2 Execute the runInstaller to upgrade the GI
GI tools or commands cannot be run until the post root script is executed. 
``` bash
[grid@node1:/worktmp/grid]$ ./runInstaller -ignorePrereq -silent -force -responseFile ~/gi.rsp 
As a root user, execute the following script(s):
	1. /u02/app/grid/12.1.0/rootupgrade.sh

Execute /u02/app/grid/12.1.0/rootupgrade.sh on the following nodes: 
[node1, node2]

Run the script on the local node first. After successful completion, you can start the script in parallel on all other nodes, 
except a node you designate as the last node. When all the nodes except the last node are done successfully, 
run the script on the last node.

Successfully Setup Software.
As install user, execute the following script to complete the configuration.
	1. /u02/app/grid/12.1.0/cfgtoollogs/configToolAllCommands RESPONSE_FILE=<response_file>

 	Note:
	1. This script must be run on the same host from where installer was run. 
	2. This script needs a small password properties file for configuration assistants that require passwords (refer to install guide documentation).
```

#### 3.3 Execute the post-script 

During the post script execution, the database services still remain online, but only one node can serve as normal, another node is out of service at that time. The details will show in the output logs. 

``` bash
[root@node1 ~]# /u02/app/grid/12.1.0/rootupgrade.sh
[root@node2 ~]# /u02/app/grid/12.1.0/rootupgrade.sh
#after execute the script on node1, the activeversion is 11.2, 
#but the softwareversion is 12.1 now
[grid@node1:/home/grid]$ crsctl query crs softwareversion
Oracle Clusterware version on node [node1] is [12.1.0.2.0]
[grid@node1:/home/grid]$ crsctl query crs activeversion
Oracle Clusterware active version on the cluster is [11.2.0.4.0]
```

You can find "2016/04/19 20:07:19 CLSRSC-482: Running command: '/u02/app/grid/12.1.0/bin/crsctl set crs activeversion'" in log file at last node to specify the script will set the activeversion in the last node. 

``` bash [when last node root script finished]
[grid@node2:/home/grid]$ crsctl query crs softwareversion
Oracle Clusterware version on node [node2] is [12.1.0.2.0]
[grid@node2:/home/grid]$ crsctl query crs activeversion
Oracle Clusterware active version on the cluster is [12.1.0.2.0]
```

Following script is executed by GRID user, only in the upgrade node:

``` bash
[grid@node1:/home/grid]$ cd $ORACLE_HOME/cfgtoollogs 
[grid@node1:/u02/app/grid/12.1.0/cfgtoollogs]$ touch cfgrsp.properties 
[grid@node1:/u02/app/grid/12.1.0/cfgtoollogs]$ vi cfgrsp.properties 
[grid@node1:/u02/app/grid/12.1.0/cfgtoollogs]$ chmod 600 cfgrsp.properties 
[grid@node1:/u02/app/grid/12.1.0/cfgtoollogs]$ ./configToolAllCommands RESPONSE_FILE=./cfgrsp.properties
```


The GI status after upgrade:

``` bash [the GI status after upgrade]
[grid@node1:/home/grid]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.CRS.dg
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.DATA.dg
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.FRA.dg
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.asm
               ONLINE  ONLINE       node1                    Started,STABLE
               ONLINE  ONLINE       node2                    Started,STABLE
ora.net1.network
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
ora.ons
               ONLINE  ONLINE       node1                    STABLE
               ONLINE  ONLINE       node2                    STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       node2                    STABLE
ora.LISTENER_SCAN2.lsnr
      1        ONLINE  ONLINE       node1                    STABLE
ora.LISTENER_SCAN3.lsnr
      1        ONLINE  ONLINE       node1                    STABLE
ora.MGMTLSNR
      1        ONLINE  ONLINE       node1                    169.254.2.91 10.10.1
                                                             0.101,STABLE
ora.cvu
      1        ONLINE  ONLINE       node2                    STABLE
ora.mgmtdb
      1        ONLINE  ONLINE       node1                    Open,STABLE
ora.node1.vip
      1        ONLINE  ONLINE       node1                    STABLE
ora.node2.vip
      1        ONLINE  ONLINE       node2                    STABLE
ora.oc4j
      1        ONLINE  ONLINE       node2                    STABLE
ora.racdb.db
      1        ONLINE  ONLINE       node1                    Open,STABLE
      2        ONLINE  ONLINE       node2                    Open,STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       node2                    STABLE
ora.scan2.vip
      1        ONLINE  ONLINE       node1                    STABLE
ora.scan3.vip
      1        ONLINE  ONLINE       node1                    STABLE
--------------------------------------------------------------------------------
```

The ASM diskgroup status:

``` sql
SQL> col name for a10
col COMPATIBILITY for a15
col DATABASE_COMPATIBILITY for a15
select name,type,state,total_mb,free_mb,COMPATIBILITY, DATABASE_COMPATIBILITY from gv$asm_diskgroup;

NAME	   TYPE   STATE 	TOTAL_MB    FREE_MB COMPATIBILITY   DATABASE_COMPAT
---------- ------ ----------- ---------- ---------- --------------- ---------------
FRA	   EXTERN MOUNTED	   20480      20332 11.2.0.0.0	    10.1.0.0.0
DATA	   EXTERN MOUNTED	   30720      28756 11.2.0.0.0	    10.1.0.0.0
CRS	   NORMAL MOUNTED	   24576      15430 11.2.0.0.0	    10.1.0.0.0
FRA	   EXTERN MOUNTED	   20480      20332 11.2.0.0.0	    10.1.0.0.0
DATA	   EXTERN MOUNTED	   30720      28756 11.2.0.0.0	    10.1.0.0.0
CRS	   NORMAL MOUNTED	   24576      15430 11.2.0.0.0	    10.1.0.0.0
```

### 4. Upgrade GI force when some nodes were inactive

When some of the nodes in the cluster are inaccessible, perform a forced upgrade is available. Execute <code>rootupgrade</code> script by root user with force option. 

``` bash
$GRID_HOME/rootupgrade -force 
```

When the inaccessible nodes become active after a forced upgrade, execute the following command in the first node to let the inaccessible node join the cluster. 

``` bash
$GRID_HOME/crs/install/rootcrs.pl -join -existNode node1 upgrade_node node2 
```

### 5. Performing upgrade RDBMS 

Although RDBMS can upgrade directly, but it's recommended install a new RDBMS binary code and then perform a upgrade command. 

#### 5.1 RDBMS software installation 

RDBMS installation response file : 

```
[oracle@node1.:/home/oracle]$ cat db.rsp |grep -v ^# |grep -v ^$
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v12.1.0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=node1.
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
SELECTED_LANGUAGES=en
ORACLE_HOME=/u02/app/oracle/product/12.1.0/db1
ORACLE_BASE=/u02/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=
oracle.install.db.BACKUPDBA_GROUP=dba
oracle.install.db.DGDBA_GROUP=dba
oracle.install.db.KMDBA_GROUP=dba
oracle.install.db.rac.configurationType=
oracle.install.db.CLUSTER_NODES=node1,node2
oracle.install.db.isRACOneInstall=false
oracle.install.db.racOneServiceName=
oracle.install.db.rac.serverpoolName=
oracle.install.db.rac.serverpoolCardinality=0
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
oracle.install.db.config.starterdb.globalDBName=
oracle.install.db.config.starterdb.SID=
oracle.install.db.ConfigureAsContainerDB=false
oracle.install.db.config.PDBName=
oracle.install.db.config.starterdb.characterSet=
oracle.install.db.config.starterdb.memoryOption=false
oracle.install.db.config.starterdb.memoryLimit=
oracle.install.db.config.starterdb.installExampleSchemas=false
oracle.install.db.config.starterdb.password.ALL=
oracle.install.db.config.starterdb.password.SYS=
oracle.install.db.config.starterdb.password.SYSTEM=
oracle.install.db.config.starterdb.password.DBSNMP=
oracle.install.db.config.starterdb.password.PDBADMIN=
oracle.install.db.config.starterdb.managementOption=DEFAULT
oracle.install.db.config.starterdb.omsHost=
oracle.install.db.config.starterdb.omsPort=0
oracle.install.db.config.starterdb.emAdminUser=
oracle.install.db.config.starterdb.emAdminPassword=
oracle.install.db.config.starterdb.enableRecovery=false
oracle.install.db.config.starterdb.storageType=
oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=
oracle.install.db.config.starterdb.fileSystemStorage.recoveryLocation=
oracle.install.db.config.asm.diskGroup=
oracle.install.db.config.asm.ASMSNMPPassword=
MYORACLESUPPORT_USERNAME=
MYORACLESUPPORT_PASSWORD=
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
DECLINE_SECURITY_UPDATES=true
PROXY_HOST=
PROXY_PORT=
PROXY_USER=
PROXY_PWD=
COLLECTOR_SUPPORTHUB_URL=
```

Install RDBMS with above response file in silent mode: 

```
[oracle@node1:/worktmp/database]$ ./runInstaller -silent -responseFile /home/oracle/db.rsp -ignorePrereq -ignoreSysPreReqs -ignoreDiskWarning
The installation of Oracle Database 12c was successful.
Please check '/u01/app/oraInventory/logs/silentInstall2016-04-20_08-58-39PM.log' for more details.
As a root user, execute the following script(s):
	1. /u02/app/oracle/product/12.1.0/db1/root.sh
Execute /u02/app/oracle/product/12.1.0/db1/root.sh on the following nodes: 
[node1, node2]
```

Execute root script as root user. 

```
[root@node1 ~]# /u02/app/oracle/product/12.1.0/db1/root.sh
[root@node2 ~]# /u02/app/oracle/product/12.1.0/db1/root.sh
```

#### 5.2 Pre-upgrade tasks 

``` bash
SQL> @/u02/app/oracle/product/12.1.0/db1/rdbms/admin/preupgrd.sql 
ACTIONS REQUIRED:
1. Review results of the pre-upgrade checks:
 /u01/app/oracle/cfgtoollogs/racdb/preupgrade/preupgrade.log
2. Execute in the SOURCE environment BEFORE upgrade:
 /u01/app/oracle/cfgtoollogs/racdb/preupgrade/preupgrade_fixups.sql
3. Execute in the NEW environment AFTER upgrade:
 /u01/app/oracle/cfgtoollogs/racdb/preupgrade/postupgrade_fixups.sql
```

Fix warnings and errors in the preupgrade.log, then execute the preupgrade_fixups.sql 

``` bash
SQL> @/u01/app/oracle/cfgtoollogs/racdb/preupgrade/preupgrade_fixups.sql
```

#### 5.3 Performing upgrade database 
Before perform upgrade, it's important to copy the database password file to new $ORACLE_HOME/dbs. 

``` bash 
[oracle@node1:/home/oracle]$ cp -p $ORACLE_HOME/dbs/orapwracdb1 /u02/app/oracle/product/12.1.0/db1/dbs/
[oracle@node2:/home/oracle]$ cp -p $ORACLE_HOME/dbs/orapwracdb2 /u02/app/oracle/product/12.1.0/db1/dbs/
```

Disable archival is also advisable, for avoiding excessive archive logs generation during upgrade. 

```
[oracle@node1:/home/oracle]$ srvctl stop database -d racdb -o immediate
[oracle@node1:/home/oracle]$ sqlplus "/as sysdba"
SQL> startup mount
SQL> alter database noarchivelog;
SQL> shutdown immediate
```

Create a legacy init file to local from spfile, modify <code>cluster_database</code> to false, do some necessary modify. I modified the ORACLE_BASE to /u02, the adump to /u02, created the adump directory, and deleted the <code>remote_listener</code>.

```
SQL> create pfile='/home/oracle/init.ora' from spfile;
```

Now, SET the new ORACLE variables to 12c, and add the following entry to /etc/oratab in both nodes:

```
[oracle@node1:/home/oracle]$ . .bash_profile
[oracle@node1:/home/oracle]$ echo $ORACLE_HOME 
/u02/app/oracle/product/12.1.0/db1
[oracle@node1:/home/oracle]$ which sqlplus
/u02/app/oracle/product/12.1.0/db1/bin/sqlplus
[root@node2 ~]# tail -2 /etc/oratab 
#racdb:/u01/app/oracle/product/11gr2:N		# line added by Agent
racdb:/u02/app/oracle/product/12.1.0/db1:N
```

Exectue startup upgrade now: 

```
[oracle@node1:/home/oracle]$ sqlplus "/as sysdba"
SQL> startup upgrade pfile='/home/oracle/init.ora';
```

But when I upgrade by manually, the ASM diskgroup cannot be mounted as encounterred "permission denied" errors. 

``` bash [Fix permission denied error of ASM disk]
[grid@node1:/home/grid]$ cd $ORACLE_HOME/bin
[grid@node1:/u02/app/grid/12.1.0/bin]$ ll /u01/app/oracle/product/11gr2/bin/oracle
-rwsr-s--x 1 oracle asmadmin 239626641 Apr 20 17:15 /u01/app/oracle/product/11gr2/bin/oracle
[grid@node1:/u02/app/grid/12.1.0/bin]$ ll /u02/app/oracle/product/12.1.0/db1/bin/oracle
-rwsr-s--x 1 oracle oinstall 323762228 Apr 20 21:06 /u02/app/oracle/product/12.1.0/db1/bin/oracle
#change the group of 12c's bin/oracle to asmadmin in both nodes
[grid@node1.:/u02/app/grid/12.1.0/bin]$ ./setasmgidwrap o=/u02/app/oracle/product/12.1.0/db1/bin/oracle
[grid@node2.:/home/grid]$ cd $ORACLE_HOME/bin
[grid@node2.:/u02/app/grid/12.1.0/bin]$ ./setasmgidwrap o=/u02/app/oracle/product/12.1.0/db1/bin/oracle
```

After changed the oracle binary attributes, the upgrade works now: 

``` sql [Execute upgrade again]
SQL> startup upgrade pfile='/home/oracle/init.ora';
ORACLE instance started.
Total System Global Area 2483027968 bytes
Fixed Size		    2927432 bytes
Variable Size		  721421496 bytes
Database Buffers	 1744830464 bytes
Redo Buffers		   13848576 bytes
Database mounted.
Database opened.
SQL> 
```

Execute upgrade utility:

``` bash
[oracle@node1:/home/oracle]$ cd $ORACLE_HOME/rdbms/admin
[oracle@node1:/u02/app/oracle/product/12.1.0/db1/rdbms/admin]$ 
[oracle@node1:/u02/app/oracle/product/12.1.0/db1/rdbms/admin]$ $ORACLE_HOME/perl/bin/perl catctl.pl -n 4 -l /tmp catupgrd.sql
```
#### 5.4 Post-upgrade tasks 

Database will automatically shutdown once the previous command successfully completes. Then I need to perform some tasks to enable CLUSTER_DATABASE, enable ARCHIVE etc.: 

```
#modify cluster_database to true, uncommet the remote_listener in the init file
SQL> startup mount pfile='/home/oracle/init.ora';
SQL> ALTER DATABASE ARCHIVELOG;
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP pfile='/home/oracle/init.ora';
SQL> create spfile='+DATA/RACDB/spfileracdb.ora' from pfile='/home/oracle/init.ora';
#add following entries to $ORACLE_HOME/dbs/initracdb1(2).ora to both nodes
SPFILE='+DATA/racdb/spfileracdb.ora'
```

Executing compile scripts:

```
SQL> execute dbms_stats.gather_fixed_objects_stats;
SQL> @?/rdbms/admin/utlrp.sql 		--recomiples all invalid objects on the database
SQL> @?/rdbms/admin/utluiobj.sql	--verfies the validity of all packages/classes on the database
SQL> @?/rdbms/admin/catuppst.sql
SQL> @?/rdbms/admin/utlu121s.sql	--displays database upgrade summary
SQL> SHUTDOWN IMMEDIATE;
```

Upgrade the database version in OCR by using the following command:

```
[oracle@node1:/home/oracle]$ srvctl upgrade database -d racdb -o /u02/app/oracle/product/12.1.0/db1
[oracle@node1:/home/oracle]$ srvctl start database -d racdb -o open
```

Run postupgrade_fixups.sql script

```
SQL> @/u01/app/oracle/cfgtoollogs/racdb/preupgrade/postupgrade_fixups.sql
```

Adjust the compatibility of ASM diskgroup's attribute "compatible.asm" and "compatible.rdbms" to 12.1 if necessary, for example, convert non-CDB to PDB, it's necessary to update these values to 12.1. For the convenience of downgrade, this step will be omitted.

``` sql [Executed under with GRID user] 
ALTER DISKGROUP data SET ATTRIBUTE 'compatible.asm' = '12.1';
ALTER DISKGROUP data SET ATTRIBUTE 'compatible.rdbms' = '12.1',
```


Upgrade the TIMEZONE file, this can refer to previous post [Upgrade Single Database to 12c](/upgrade-single-database-to-12c.html).   

Confirm the tnsnames file and  the listener file in the new Oracle 12c home are correct, if necessary, copy them from 11g oracle home. 

#### 5.5 Detach 11g grid and rdbms home 

This step is optional, or if the new 12c is stable for a long time, you need to deinstall the 11G software to save space, you can detach the relative home, and then deinstall them. 

- Detach 11g grid home in node1 and node2 

``` bash
[grid@node1:/home/grid]$ /u01/app/11gr2/grid/oui/bin/runInstaller -silent \
-detachHome -invPtrLoc /etc/oraInst.loc ORACLE_HOME="/u01/app/11gr2/grid" \
ORACLE_HOME_NAME="Ora11g_gridinfrahome1" CLUSTER_NODES="{node1,node2}" -local
```

- Detach 11g rdbms home in node1 and node2 

```
[oracle@node1:/home/oracle]$ /u01/app/oracle/product/11gr2/oui/bin/runInstaller \
-silent -detachHome -invPtrLoc /etc/oraInst.loc \
ORACLE_HOME="/u01/app/oracle/product/11gr2" \
ORACLE_HOME_NAME="OraDb11g_home1" CLUSTER_NODES="{node1,node2}" -local
```

- Verify the detach result 

```
$GRID_HOME/OPatch/opatch lsinventory -oh $OLD_GRID_HOME
$ORACLE_HOME/OPatch/opatch lsinventory -oh $OLD_ORACLE_HOME
```


### 6. Downgrade RDBMS

Disable the CLUSTER_DATABASE initialization parameter by editing the legacy init file, and stop the database as follows:

```
SQL> create pfile='/home/oracle/init.ora' from spfile;
[oracle@node1:/home/oracle]$ srvctl stop database -d racdb
#Modify init file to meet the downgrade requirements
SQL> startup downgrade pfile='/home/oracle/init.ora';
SQL> SPOOL /tmp/dbdowngrade.log
SQL> @?/rdbms/admin/catdwgrd.sql
```

The catdwgrd.sql script under the Oracle 12c /rdbms/admin home downgrades the 12c Database components to the previous release. If encounter any ORA issues during the course of downgrade, fix the issue and rerun the downgrade script once again.   

After complete this script, shutdown the database, and set the ORACLE variables to 11g's:

```
SQL> shutdown immediate
[oracle@node1:/home/oracle]$. .bash_profile_11g
[oracle@node1:/home/oracle]$ which sqlplus
/u01/app/oracle/product/11gr2/bin/sqlplus
#Startup in upgrade mode to reload the 11g components by running 11g ORACLE_HOME catrelod.sql
SQL> startup upgrade pfile='/home/oracle/init.ora'
SQL> spool /tmp/dbdowngrade.log
SQL> @?/rdbms/admin/catrelod.sql
```
If you previously installed a recent version of the time zone file and used the DBMS_DST PL/SQL package to upgrade TIMESTAMP WITH TIME ZONE data to that version, then install the same version of the time zone file in the release to which you are downgrading. Or you will get the following errors. 

```
SELECT TO_NUMBER('MUST_BE_SAME_TIMEZONE_FILE_VERSION')
ERROR at line 1:
ORA-01722: invalid number
```

Try to copy the 12c timezone file to 11g, and re-run the catrelod.sql. 

```
#confirm current tz file
SQL> select * from  V$TIMEZONE_FILE;
#copy the 12c timezone file to 11g
[oracle@node1:/home/oracle]$ cp /u02/app/oracle/product/12.1.0/db1/oracore/zoneinfo/timezone_18.dat \
$ORACLE_HOME/oracore/zoneinfo/
[oracle@node1:/home/oracle]$ cp /u02/app/oracle/product/12.1.0/db1/oracore/zoneinfo/timezlrg_18.dat \
$ORACLE_HOME/oracore/zoneinfo/
```

Enable the CLUSTER_DATABASE by edit the init file, create a spfile from pfile, and then startup the database, recompile all invalid objects:

```
#Edit init.ora file to enable CLUSTER_DATABASE=true
SQL> create spfile='+DATA/RACDB/spfileracdb.ora' from pfile='/home/oracle/init.ora';
SQL> shutdown immediate
SQL> startup 
SQL> @?/rdbms/admin/utlrp.sql
```

Downgrade the database version in OCR using srvctl in 12c oracle database home:

```
[oracle@node1:/home/oracle]$ /u02/app/oracle/product/12.1.0/db1/bin/srvctl downgrade \
database -d racdb -o /u01/app/oracle/product/11gr2 -t 11.2.0.4.0
```

Verify the downgrade result:

```
[oracle@node1.oraclema.com:/home/oracle]$ srvctl config database -d racdb -a 
Database unique name: racdb
Database name: racdb
Oracle home: /u01/app/oracle/product/11gr2
Oracle user: oracle
Spfile: +DATA/RACDB/spfileracdb.ora
Domain: 
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: racdb
Database instances: racdb1,racdb2
Disk Groups: DATA
Mount point paths: 
Services: 
Type: RAC
Database is enabled
Database is administrator managed
#Verify the database components
SQL> set pagesize 9999
set line 200
col comp_name for a50
col version for a15    
col status for a10
select comp_name,version,status from dba_registry;
COMP_NAME					   VERSION	   STATUS
-------------------------------------------------- --------------- ----------
OWB						   11.2.0.4.0	   VALID
Oracle Application Express			   3.2.1.00.12	   VALID
Spatial 					   11.2.0.4.0	   VALID
Oracle Multimedia				   11.2.0.4.0	   VALID
Oracle XML Database				   11.2.0.4.0	   VALID
Oracle Text					   11.2.0.4.0	   VALID
Oracle Workspace Manager			   11.2.0.4.0	   VALID
Oracle Database Catalog Views			   11.2.0.4.0	   VALID
Oracle Database Packages and Types		   11.2.0.4.0	   VALID
JServer JAVA Virtual Machine			   11.2.0.4.0	   VALID
Oracle XDK					   11.2.0.4.0	   VALID
Oracle Database Java Packages			   11.2.0.4.0	   VALID
OLAP Analytic Workspace 			   11.2.0.4.0	   VALID
Oracle OLAP API 				   11.2.0.4.0	   VALID
Oracle Real Application Clusters		   11.2.0.4.0	   VALID
```

Shutdown the database, and startup by using srvctl tools in 11g oracle database home: 

```
SQL> shutdown immediate
[oracle@node1:/home/oracle]$ srvctl start database -d racdb -o open
```

### 7. Downgrade GI 
For downgrade 12c GI, run new 12c <code>$GI_HOME/crs/install/rootcrs.sh -downgrade</code> with root user at all nodes one by one, after all nodes successfully executing preceding command, back to the first node, run <code>$GI_HOME/crs/install/rootcrs.sh -downgrade -lastnode</code>.  The <code>-force</code> option can be used when a failed upgrade happened. Run this command from a directory that has write permissions for the Oracle Grid Infrastructure installation user.

``` bash [This process will downgrade the OCR and set to the previous release]
#At node1
[root@node1 ~]# cd /home/grid
[root@node1 ~]# /u02/app/grid/12.1.0/crs/install/rootcrs.sh -downgrade
#At node2
[root@node2 ~]# cd /home/grid
[root@node2 ~]# /u02/app/grid/12.1.0/crs/install/rootcrs.sh -downgrade 
#At node1
[root@node1 ~]# cd /home/grid
[root@node1 ~]# /u02/app/grid/12.1.0/crs/install/rootcrs.sh -downgrade -lastnode
```

Running the following script on the nodes where the preceding rootcrs.sh has run successfully, this script must be run as GI user, and from 12c oracle GI home:  

``` bash
[grid@node1:/home/grid]$ /u02/app/grid/12.1.0/oui/bin/runInstaller -nowait -waitforcompletion \
-ignoreSysPrereqs -updateNodeList -silent CRS=false ORACLE_HOME=/u02/app/grid/12.1.0
[grid@node2:/home/grid]$ /u02/app/grid/12.1.0/oui/bin/runInstaller -nowait -waitforcompletion \
-ignoreSysPrereqs -updateNodeList -silent CRS=false ORACLE_HOME=/u02/app/grid/12.1.0
```

After successfully preceding commands, as the root user, bring up the previous release cluster stack using the following command on each node in the cluster:

```
[root@node1 grid]# /u01/app/11gr2/grid/bin/crsctl start crs
[root@node2 grid]# /u01/app/11gr2/grid/bin/crsctl start crs
```


####Below script can be used to delete the installation environment directly 

``` bash [Please do not do this in production]
rm -rf /etc/oraInst.loc /etc/oratab /etc/ohasd
rm -rf /tmp/.oracle /var/tmp/.oracle
rm -rf /etc/oracle /opt/ORCLfmap/
rm -rf /etc/rc.d/init.d/ohasd
rm -rf /var/lock/subsys/ohasd
rm -rf /usr/local/bin/coraenv
rm -rf /usr/local/bin/dbhome
rm -rf /usr/local/bin/oraenv
rm -f /etc/init.d/init.cssd
rm -f /etc/init.d/init.crs
rm -f /etc/init.d/init.crsd
rm -f /etc/init.d/init.evmd
rm -rf /tmp/ora* /tmp/CVU* /tmp/Ora*
rm -rf /u01/app/*
rm -rf /u02/app/*
rm -f /etc/init.d/init.ohasd
dd if=/dev/zero of=/dev/sdb bs=4096 count=1
dd if=/dev/zero of=/dev/sdc bs=4096 count=1
dd if=/dev/zero of=/dev/sdd bs=4096 count=1
dd if=/dev/zero of=/dev/sde bs=4096 count=1
dd if=/dev/zero of=/dev/sdf bs=4096 count=1
chown -R grid:oinstall /u01 /u02
chmod -R 775 /u01 /u02
```

Reference:   
[How to Upgrade to Oracle Grid Infrastructure 12c Release 1](https://docs.oracle.com/database/121/CWLIN/procstop.htm#CWLIN10001)

</br><b>EOF</b>






















