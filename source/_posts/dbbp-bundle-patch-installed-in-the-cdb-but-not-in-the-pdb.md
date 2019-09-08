---
title: dbbp bundle patch installed in the cdb but not in the pdb
categories: oracle
comments: false
date: 2019-08-16 14:37:03
tags: [patch, multitenant]
---

从远程数据库复制过来后，由于DBBP版本不一致，开启PDB的时候出现错误，且PDB处于restricted模式:
```sql
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 SLDB1			  READ WRITE YES
```
查询`pdb_plug_in_violations`
```sql
MESSAGE 				 STATUS 	    ACTION
---------------------------------------- ------------------ ----------------------------------------
DBBP bundle patch 190115 (DATABASE BUNDL PENDING	    Call datapatch to install in the PDB or
E PATCH 12.1.0.2.190115): Installed in t		    the CDB
he PDB but not in the CDB.

DBBP bundle patch 190416 (DATABASE BUNDL PENDING	    Call datapatch to install in the PDB or
E PATCH 12.1.0.2.190416): Installed in t		    the CDB
he CDB but not in the PDB.

SQL patch ID/UID 28790654/22620251 (Data PENDING	    Call datapatch to install in the PDB or
base PSU 12.1.0.2.190115, Oracle JavaVM 		    the CDB
Component (JAN2019)): Installed in the P
DB but not in the CDB.
```

* 相关版本

Linora为源库，是non-cdb; db2srv是目标库，是cdb
```sql
 /u01/app/oracle/product/12c/db_1  (opatch version 12.2.0.1.17)
------------------------------
  Patch ID  |     linora     |
------------------------------
  28731800  |        -       | Database Bundle Patch : 12.1.0.2.190115 (28731800)
  28790654  |        -       | Database PSU 12.1.0.2.190115, Oracle JavaVM Component (JAN2019)
------------------------------

  /u02/app/oracle/product/12.1.0.2/db_1  (opatch version 12.2.0.1.17)
------------------------------
  Patch ID  |     db2srv     |
------------------------------
  29141038  |        -       | Database Bundle Patch : 12.1.0.2.190416 (29141038)
  29251241  |        -       | Database PSU 12.1.0.2.190416, Oracle JavaVM Component (APR2019)
```

* datapatch衍生问题

根据`pdb_plug_in_violations`提示，执行datapatch, 结果因为两边jvm版本不一致，发生如下错误:
有些情况下，需要将CDB/PDB启动到startup upgrade模式下，才能进行datapatch

```sql
cd $ORACLE_HOME/Opatch
./datapatch -verbose

Adding patches to installation queue and performing prereq checks...
Installation queue:
  For the following PDBs: CDB$ROOT PDB$SEED SLDB1
    Nothing to roll back
    The following patches will be applied:
      29251241 (Database PSU 12.1.0.2.190416, Oracle JavaVM Component (APR2019))
  For the following PDBs: SLDB2
    The following patches will be rolled back:
      28790654 (Database PSU 12.1.0.2.190115, Oracle JavaVM Component (JAN2019))
    The following patches will be applied:
      29251241 (Database PSU 12.1.0.2.190416, Oracle JavaVM Component (APR2019))
      29141038 (DATABASE BUNDLE PATCH 12.1.0.2.190416)

Validating logfiles...
Patch 28790654 rollback (pdb SLDB2): WITH ERRORS
  logfile: /u02/app/oracle/cfgtoollogs/sqlpatch/28790654/22620251/28790654_rollback_CDB12C_SLDB1_2019Aug16_14_10_11.log (errors)
    Error at line 595: ORA-29548: Java system class reported: could not identify release specified in
    Error at line 627: ORA-29548: Java system class reported: could not identify release specified in
    Error at line 635: ORA-29548: Java system class reported: could not identify release specified in
Patch 29251241 apply (pdb SLDB2): WITH ERRORS
  logfile: /u02/app/oracle/cfgtoollogs/sqlpatch/29251241/22839506/29251241_apply_CDB12C_SLDB1_2019Aug16_14_10_32.log (errors)
    Error at line 663: ORA-29548: Java system class reported: could not identify release specified in
    Error at line 695: ORA-29548: Java system class reported: could not identify release specified in
    Error at line 703: ORA-29548: Java system class reported: could not identify release specified in
```

此时查询pdb的`dba_registry_sqlpatch`, 会发现database的补丁已经应用，但是OJVM因为上述错误无法应用。
```sql
SQL> alter session set container = sldb2;

ACTION_TIME	     ACTION	STATUS	   DESCRIPTION					 VERSION      PATCH_ID BUNDLE_SER
-------------------- ---------- ---------- --------------------------------------------- ---------- ---------- ----------
16-AUG-2019 09:18:16 APPLY	SUCCESS    DATABASE BUNDLE PATCH 12.1.0.2.190115	 12.1.0.2     28731800 DBBP
16-AUG-2019 09:24:23 APPLY	SUCCESS    Database PSU 12.1.0.2.190115, Oracle JavaVM C 12.1.0.2     28790654
					   omponent (JAN2019)

16-AUG-2019 15:38:08 ROLLBACK	WITH ERROR Database PSU 12.1.0.2.190115, Oracle JavaVM C 12.1.0.2     28790654
				S	   omponent (JAN2019)

16-AUG-2019 15:38:08 APPLY	WITH ERROR Database PSU 12.1.0.2.190416, Oracle JavaVM C 12.1.0.2     29251241
				S	   omponent (APR2019)

16-AUG-2019 15:38:09 APPLY	SUCCESS    DATABASE BUNDLE PATCH 12.1.0.2.190416	 12.1.0.2     29141038 DBBP
```
MOS查询出来两个bug：
Bug 17610418 - ORA-29548 from plugged in different version non CDB as PDB [17610418.8]
Bug 16986421 - ORA-29548 from plug in of lower version PDB with Java [16986421.8]
但是这个两个bug是在12.1.0.2 server patch set已经fixed，已经找不到单独的patch。

* 解决方案

尝试在复制前，将目标库的OJVM补丁回退到原始状态，等PDB复制过来后，再进行补丁的安装，问题得到解决.

## 总结
* 1. PDB出现DBBP相关的错误时，查询CDB及PDB的`dba_registry_sqlpatch`，比较两者的区别
* 2. 根据上述结果，评估是否可以进行datapatch
* 3. 在进行远程复制的时候，如果两边OJVM版本不一致，可以考虑现在目标端回滚OJVM，再进行远程复制


</br>
__Reference__:
[After running datapatch, PDB plugin or cloned db returns violations shown in PDB_PLUG_IN_VIOLATION (Doc ID 1635482.1)](
https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=520987474833173&id=1635482.1&_afrWindowMode=0&_adf.ctrl-state=15i0wfigbx_300)


__EOF__

