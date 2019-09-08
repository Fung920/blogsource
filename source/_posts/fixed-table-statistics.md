---
title: Fixed Table统计信息
categories: oracle
comments: false
date: 2019-08-08 00:11:00
tags: maintenance
---

Oracle有大量的内部视图供DBA使用，这些视图底层表以X$开头，Fixed Objects指的是这些以x$开头的表(下文中称为基表)及它们的索引。
很多v$开头的视图基表都是x$表，包括动态性能视图，管理视图，如dba_free_space等，因此，这些fixed objects的统计信息就显得极其重要。

## 1. Fixed objects统计信息重要性
优化器在生成执行计划的时候依赖于这些基表的统计信息，如果这些基表的统计信息缺失，不像用户对象的统计信息缺失，Oracle会使用dynamic sampling，优化器在这些缺失统计信息的基表上，会使用预设的默认值进行执行计划的评估。在这种情况下，执行计划可能是极其糟糕的。因此，可能存在查询某些动态性能视图或者数据字典时，出现很慢的情况。

例如, `X$KTFBUE`记录了数据文件、extent的位图等信息，在查询dba_free_space的时候，会使用到这个基表，如果这个表的统计信息为0，则有可能导致查询dba_free_space很慢. 当这个基表缺失统计信息时，该表的行数默认为10万行。

```sql
column database_creation format a18
column last_analyzed format a18
select dbid
      ,to_char(created,'dd.mm.yyyy hh24:mi') database_creation
      ,version
      ,(select to_char(max(last_analyzed),'dd.mm.yyyy hh24:mi') last_analyzed
         from dba_tab_statistics
         where object_type='FIXED TABLE') last_analyzed
from v$database,v$instance;
```

* 如何收集
   在12c以前，基表的统计信息是不会通过Oracle自动任务去收集的，需要手动执行下面这个procedure, 用户需要有sysdba或者`ANALYZE ANY DICTIONARY`的权限。
```sql
exec DBMS_STATS.GATHER_FIXED_OBJECTS_STATS;
```
   - 上面这个procedure与`DBMS_STATS.GATHER_TABLE_STATS`的区别是，它不会收集表/索引的blocks信息，因为基表是存放在内存中随时动态变化的，它们的blocks永远设置为0.
   - __统计信息的收集都是会消耗资源，不建议在业务高峰期对任何批量对象进行统计信息的收集__。
   -  12c以后，虽然自动任务窗口会收集基表统计信息，但是，其限制是在窗口时间内，其优先级是最低的，要先等到用户对象的统计信息收集完，等数据字典的统计信息收集完，同时这些基表的统计信息不存在，即是说，自动窗口不会去更新基表统计信息的；因此，建议定期手工对基表进行收集。

   尽管基表生存周期是在内存中，但其统计信息是会保存在磁盘中，因此，实例重启后，除非负载有很大的变化，并没有必要重新收集统计信息。


## 2. Fixed tables统计信息对数据库的影响
对dba_extents, v$access, V$RMAN_BACKUP_JOB_DETAILS, V$RMAN_STATUS,DBA_FREE_SPACE等视图有很大影响，很多时候查询这些视图很慢，极大可能就是因为基表统计信息缺失或者存在错误的统计信息。

</br>
Reference:
[How to Gather Statistics on Objects Owned by the 'SYS' User and 'Fixed' Objects (Doc ID 457926.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=hy44rkzga_65&id=2331567.1&_afrLoop=463004944185813)
[Fixed Objects Statistics (GATHER_FIXED_OBJECTS_STATS) Considerations (Doc ID 798257.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=463163420960672&parent=DOCUMENT&sourceId=2331567.1&id=798257.1&_afrWindowMode=0&_adf.ctrl-state=hy44rkzga_122)
[Best Practices for Gathering Optimizer Statistics with Oracle Database 12c Release 2](https://www.oracle.com/technetwork/database/bi-datawarehousing/twp-bp-for-stats-gather-12c-1967354.pdf)
[ORA-01555 Caused By Auto Execute Of Job "SYS"."PMO_DEFERRED_GIDX_MAINT_JOB" (Doc ID 2523018.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=465970265177832&id=2523018.1&_afrWindowMode=0&_adf.ctrl-state=hy44rkzga_293)
[Database SQL Tuning Guide - Gathering Statistics for Fixed Objects](https://docs.oracle.com/database/121/TGSQL/tgsql_stats.htm#TGSQL95110)

__EOF__
