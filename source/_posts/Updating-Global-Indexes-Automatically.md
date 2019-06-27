---
title: Updating Global Indexes Automatically
categories: oracle
comments: false
date: 2019-06-26 10:38:29
tags:
---

By default, partition table maintenance such as drop/truncate partitions invalidate corresponding global index which mark them as `UNUSABLE`. User must rebuild the corresponding indexes. Database lets user override the default behavior by specifying `update indexes` clause. With this option, database will update the indexes at the same time it executes the maintenance  DDL statements,  and not mark them as `UNUSABLE`.

Prior to 12c, update indexes is a time consuming operation, DBA must wait for the index rebuild complete.

<!--more-->

As of 12c, a new feature is supported by Oracle database — Asynchronous Global Index Maintenance. 

> The partition maintenance operations `DROP` `PARTITION` and `TRUNCATE` `PARTITION` are optimized by making the index maintenance for metadata only.
>
> Asynchronous global index maintenance for `DROP` and `TRUNCATE` is performed by default; however, the `UPDATE` `INDEXES` clause is still required for backward compatibility.

# 1. Limitations of asynchronous global index maintenance

* Only performed on the heap tables

* No support for tables with object types

* No support for tables with domain indexes

* Not performed for the user SYS

  

# 2. Scheduler Jobs

There's an automatically maintenance scheduler job `SYS.PMO_DEFERRED_GIDX_MAINT_JOB` to clean up all global indexes. This job is schedule at 2:00AM by defaul. You can run this job at any time by calling `DBMS_SCHEDULER.RUN_JOB`,also, DBA can modify scheduler window for running this job.

```sql
  -- query the maintenance window
  select job_name , start_date,enabled,state,comments
  from dba_scheduler_jobs
  where job_name ='PMO_DEFERRED_GIDX_MAINT_JOB';
  
  -- execute the job manually
  exec dbms_scheduler.run_job('SYS.PMO_DEFERRED_GIDX_MAINT_JOB')

  -- query jobs running status
  select job_name, start_date, enabled, state, comments
   from dba_scheduler_jobs
   where job_name ='PMO_DEFERRED_GIDX_MAINT_JOB';

   select * from dba_scheduler_job_run_details
   where job_name ='PMO_DEFERRED_GIDX_MAINT_JOB';
```

# 3. Summary

  If we truncate/drop a partition tables with `update indexes`, we can maintenance index manually:

  * Check the index status

    ```sql
    SELECT table_name, index_name,
           orphaned_entries,status
    FROM   user_indexes
    ORDER BY 1;
    ```

    

  * Execute one of the following SQLs

    ```sql
    /* This PL/SQL package procedure gathers 
    the list of global indexes in the system that
    may require cleanup and runs the operations necessary
    to restore the indexes to a clean state.
    */
    exec DBMS_PART.CLEANUP_GIDX('USERNAME','INDEX_NAME'); -- specific index
    --database level
    exec dbms_part.cleanup_gidx
    
    --schema level
    exec dbms_part.cleanup_gidx(<schema_name>);
    
    --table level
    exec dbms_part.cleanup_gidx(<schema_name>, <table_name>);
    ```

    ```
    -- this SQL statement rebuilds the entire index or index partition as was done prior to Oracle Database 12.1 releases
    ALTER INDEX INDEX_NAME REBUILD;
    
    --This SQL statement cleans up any orphaned entries in index blocks
    ALTER INDEX  INDEX_NAME COALESCE CLEANUP;
    ```

  * Check the index status again




__EOF__
