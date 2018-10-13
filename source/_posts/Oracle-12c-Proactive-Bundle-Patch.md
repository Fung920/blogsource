---
title: Oracle 12c Proactive Bundle Patch
categories: oracle
comments: false
date: 2018-10-10 07:20:39
tags:
   - patch
   - 12c
---
As of Oracle 12.1.0.2, ORACLE provide a new way to patch PSU which called Proactive Bundle Patch(DBBP) and replace the previous PSU.
With DBBP, applying patch is more easier. PSU is conflict with DBBP, if you want to apply DBBP in your database which PSUs are installed, you have to rollback all PSUs to make sure applying DBBP successfully.
Because DBBP covers more bug fixes than PSU, it's recommended to use DBBP to apply latest patches.
<!--more-->
# 1. Pre-installation task
First, it's always essential to check the `opatch` version, find the readme file in the patchset to meet the minimum version. In RAC environment, both `GRID_HOME` and `ORACLE_HOME` Opatch should be replaced with target version.
* Reserve historical information

```sql
<ORACLE_HOME>/OPatch/opatch lsinventory -detail -oh <ORACLE_HOME> >/tmp/lsinv.info
<ORACLE_HOME>/OPatch/opatch lspatches >> /tmp/lsinv.info
SQL> SET LINESIZE 200
COLUMN action_time FORMAT A20
COLUMN action FORMAT A10
COLUMN status FORMAT A10
COLUMN description FORMAT A40
COLUMN version FORMAT A10
COLUMN bundle_series FORMAT A10

spool /tmp/datapatch.info
SELECT TO_CHAR(action_time, 'DD-MON-YYYY HH24:MI:SS') AS action_time,
action, status, description, version, patch_id, bundle_series
FROM   sys.dba_registry_sqlpatch
ORDER by action_time;
spool off
```

* Run OPatch Conflict check

```sh
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir <UNZIPPED_PATCH_LOCATION>/27968010/27547374
```

* One-off patch conflict detection and resolution

```sh
GRID_HOME/OPatch/opatchauto apply <UNZIPPED_PATCH_LOCATION>/27968010 -analyze
```

# 2. Installation
## 2.1 Applying DBBP
* opatchauto
Running below commands on all nodes, this command require sequential execution, __DO NOT run in both nodes parallel in RAC__.

```sh
--with root user
export PATH=$PATH:<GI_HOME>/OPatch
opatchauto apply <UNZIPPED_PATCH_LOCATION>/27968010
```
After `opatchauto` installed, `lspatches` will show latest patch numbers, but they're not yet applied to database.
## 2.2 Datapatch
Execute below commands in one of instance(RAC) with Oracle user id:
```sql
--for CDB
alter pluggable database all open;
$ORACLE_HOME/datapatch -verbose
SQL> @?/rdbms/admin/utlrp.sql
--If an OJVM PSU is installed or planned to be installed, no further actions are necessary. Otherwise, the workaround of using the OJVM Mitigation patch can be activated. As SYSDBA do the following from the admin directory
SQL> @?/rdbms/admin/dbmsjdev.sql
SQL> exec dbms_java_dev.disable
```

* Upgrade rman catalog if any

```sql
rman catalog username/password@alias
RMAN> UPGRADE CATALOG;
```

## 2.3 Verifying patching result
```sql
ET LINESIZE 400

COLUMN action_time FORMAT A20
COLUMN action FORMAT A10
COLUMN status FORMAT A10
COLUMN description FORMAT A40
COLUMN version FORMAT A10
COLUMN bundle_series FORMAT A10

SELECT TO_CHAR(action_time, 'DD-MON-YYYY HH24:MI:SS') AS action_time,
       action,
       status,
       description,
       version,
       patch_id,
       bundle_series
FROM   sys.dba_registry_sqlpatch
ORDER by action_time;
```

# 3. Backout
* Execute as root to uninstall patches

```sh
--RAC
<GI_HOME>/OPatch/opatchauto rollback <UNZIPPED_PATCH_LOCATION>/27968010
--Non RAC
opatch rollback -id 27547374
```

* Update data dictionary with datapatch

```sql
SQL> startup
--CDB, not necessary if non-CDB
SQL> alter pluggable database all open;
$ORACLE_HOME/OPatch/datapatch -verbose
```
# 4. Trouble shooting
In Linux platform, with error:
```
opatchauto/opatch apply failing with CLSRSC-46: Error: '<GRID_HOME>/suptools/tfa/release/tfa_home/jlib/jewt4.jar' does not exist
```
According to MOS: [opatchauto/opatch apply failing with CLSRSC-46: Error: '<GRID_HOME>/suptools/tfa/release/tfa_home/jlib/jewt4.jar' does not exist (Doc ID 2409411.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=331630215485049&id=2409411.1&_adf.ctrl-state=v1xoqgu3h_810), following the instructions:
```sh
cp -p $GRID_HOME/crs/sbs/crsconfig_fileperms.sbs $GRID_HOME/crs/sbs/crsconfig_fileperms.sbs.bak
cp -p $GRID_HOME/crs/utl/<node>/crsconfig_fileperms $GRID_HOME/crs/utl/<node>/crsconfig_fileperms.bak
```
Remove below lines in above files:
```sh
unix %ORA_CRS_HOME%/suptools/tfa/release/tfa_home/jlib/jdev-rt.jar %HAS_USER% %ORA_DBA_GROUP% 0644
unix %ORA_CRS_HOME%/suptools/tfa/release/tfa_home/jlib/jewt4.jar %HAS_USER% %ORA_DBA_GROUP% 0644
```

Run `opatchauto resume` again will proceed to apply patches.

* AIX `dba_registry_sqlpatch` shows nothing

This is related with [BUG 20244108 - QOPIPREP.BAT MODIFIES XML INVENTORY WHILE READING], after applying one-off patch [Patch 20244108](https://support.oracle.com/epmos/faces/PatchDetail?requestId=18676540&_afrLoop=335757918424614&patchId=20244108&_afrWindowMode=0&_adf.ctrl-state=tcbuacddo_72) everything works fine.


__EOF__
