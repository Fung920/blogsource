---
title: Tablespace in backup pending after DB2 load
categories: db2
comments: false
date: 2019-03-18 21:19:13
tags: db2load
---

After enabling archive log mode, customer complaint their database cannot DML any more.
After examining db2diag.log and tablespace status, one of tablespaces used by db2 load are in `backup pending` status.

From the inforcenter of load command, load have three options which would impact database status in different log circulate mode.

* __COPY NO__
     Specifies that the table space in which the table resides will be placed in backup pending state if forward recovery is enabled (that is, logretain or userexit is on). The COPY NO option will also put the table space state into the Load in Progress table space state. This is a transient state that will disappear when the load completes or aborts. The data in any table in the table space cannot be updated or deleted until a table space backup or a full database backup is made. However, it is possible to access the data in any table by using the SELECT statement.
     ___LOAD with COPY NO on a recoverable database leaves the table spaces in a backup pending state___. For example, performing a LOAD with COPY NO and INDEXING MODE DEFERRED will leave indexes needing a refresh. Certain queries on the table might require an index scan and will not succeed until the indexes are refreshed. The index cannot be refreshed if it resides in a table space which is in the backup pending state. In that case, access to the table will not be allowed until a backup is taken. Index refresh is done automatically by the database when the index is accessed by a query. ___If one of COPY NO, COPY YES, or NONRECOVERABLE is not specified, and the database is recoverable (logretain or logarchmeth1 is enabled), then COPY NO is the default___.


* __COPY YES__
    Specifies that a copy of the loaded data will be saved. This option is invalid if forward recovery is disabled.
   - _USE TSM_
      Specifies that the copy will be stored using Tivoli® Storage Manager (TSM).
   - _OPEN num-sess SESSIONS_
      The number of I/O sessions to be used with TSM or the vendor product. The default value is 1.
   - _TO device/directory_
      Specifies the device or directory on which the copy image will be created.
   - _LOAD lib-name_
      The name of the shared library (DLL on Windows operating systems) containing the vendor backup and restore I/O functions to be used. It can contain the full path. If the full path is not given, it will default to the path where the user exit programs reside.


* __NONRECOVERABLE__
    Specifies that the load transaction is to be marked as nonrecoverable and that it will not be possible to recover it by a subsequent roll forward action. The roll forward utility will skip the transaction and will mark the table into which data was being loaded as "invalid". The utility will also ignore any subsequent transactions against that table. After the roll forward operation is completed, such a table can only be dropped or restored from a backup (full or table space) taken after a commit point following the completion of the non-recoverable load operation.
   ___With this option, table spaces are not put in backup pending state following the load operation, and a copy of the loaded data does not have to be made during the load operation. If one of COPY NO, COPY YES, or NONRECOVERABLE is not specified, and the database is not recoverable (logretain or logarchmeth1 is not enabled), then NONRECOVERABLE is the default___.


From the explanation, impacted database because in non-archive log mode, therefore default is `nonrecoverabel` option, after enabling archive log, the default behavior is change to `copy no`, which  would put database/tablespace in backup pending mode.

With customer's environment, they don't want to change the load command. But no worries, there's a db2set registry `DB2_LOAD_COPY_NO_OVERRIDE` can achive this.

This registry also have three options that identical to  load command: `copy no`, `copy yes` and `nonrecoverable`, after setting this registry to `nonrecoverable`(no recycle will be needed), issue is gone.
This registry will be ignored in HADR standby database. Only applicable for primary database.

__But because `nonrecvoerable` will not recover loaded table from redo log, it's highly recommended to take a full backup after load__.

__EOF__

Reference: [LOAD command](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_9.7.0/com.ibm.db2.luw.admin.cmd.doc/doc/r0008305.html)
           [DB2 registry and environment variables](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_9.7.0/com.ibm.db2.luw.admin.regvars.doc/doc/r0005669.html#r0005669__M_DB2_LOAD_COPY_NO_OVERRIDE)
