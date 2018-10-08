---
layout: post
title: "管理AWR快照"
date: 2014-07-17 10:52:12
comments: false
categories: oracle
tags: tuning
keywords: awr
description: how to manage Automatic Workload Repository
---
<h3>1.管理快照</h3>
Oracle默认每小时生成一次快照，11g中，默认保存时间为8天，而在10g中，默认保存时间为7天。Oracle提供了一个<code>DBMS_WORKLOAD_REPOSITORY</code>包用于管理快照，包括创建删除修改快照，但必须要有DBA权限才能进行调用。
<!--more-->
##### 查看当前快照设定
```
SYS@linora> col SNAP_INTERVAL for a20
SYS@linora> col RETENTION for a20
SYS@linora> select * from dba_hist_wr_control;

      DBID SNAP_INTERVAL        RETENTION            TOPNSQL
---------- -------------------- -------------------- ----------
3385851293 +00000 01:00:00.0    +00008 00:00:00.0    DEFAULT
```
##### 修改快照为15分钟一次，保存10天
```
SYS@linora> BEGIN
  2  DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(
  3    interval  =>  15,
  4    retention =>  10*24*60);
  5  END;
  6  /
PL/SQL procedure successfully completed.
```
在这个包中，还有其他选项，如：
```
BEGIN
  DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(retention => 43200,
                                                    interval  => 30,
                                                    topnsql   => 100,
                                                    dbid      => 3310949047);
END;
/
```
1.retention表示快照保留时间，以分钟计算，最小值为1天，最大为100年。如果此参数为0，表示永久保留。  
2.interval表示快照生产间隔时间，以分钟计算，最小10分钟，最大为1年。如果此参数为0，则表示禁用awr。  
3.topnsql表示抓取符合以下每一项条件的top N sql：
```
1. Elapsed Time (ms)
2. CPU Time (ms)
3. Executions
4. Buffer Gets
5. Disk Reads
6. Parse Calls
7. Rows
8. User I/O Wait Time (ms)
9 Cluster Wait Time (ms)
10. Application Wait Time (ms)
11. Concurrency Wait Time (ms)
12. Invalidations
13. Version Count
14. Sharable Mem(KB)
```
##### 创建快照
```
SYS@linora> col BEGIN_INTERVAL_TIME for a50 
SYS@linora> col FLUSH_ELAPSED for a30
SYS@linora> select SNAP_ID,BEGIN_INTERVAL_TIME,FLUSH_ELAPSED from dba_hist_snapshot order by snap_id;
   SNAP_ID BEGIN_INTERVAL_TIME                                FLUSH_ELAPSED
---------- -------------------------------------------------- ------------------------------
        98 17-JUL-14 02.15.08.645 PM                          +00000 00:00:00.6
        99 17-JUL-14 02.30.11.373 PM                          +00000 00:00:00.6
SYS@linora> exec dbms_workload_repository.create_snapshot();
PL/SQL procedure successfully completed.
SYS@linora>  select SNAP_ID,BEGIN_INTERVAL_TIME,FLUSH_ELAPSED from dba_hist_snapshot order by snap_id;
   SNAP_ID BEGIN_INTERVAL_TIME                                FLUSH_ELAPSED
---------- -------------------------------------------------- ------------------------------
        98 17-JUL-14 02.15.08.645 PM                          +00000 00:00:00.6
        99 17-JUL-14 02.30.11.373 PM                          +00000 00:00:00.6
       100 17-JUL-14 02.45.14.378 PM                          +00000 00:00:01.4
```
##### 删除快照
```
SYS@linora> select SNAP_ID,BEGIN_INTERVAL_TIME,FLUSH_ELAPSED from dba_hist_snapshot 
where snap_id>=75 and snap_id <=80;
   SNAP_ID BEGIN_INTERVAL_TIME                                FLUSH_ELAPSED
---------- -------------------------------------------------- ------------------------------
        75 16-JUL-14 10.50.34.000 AM                          +00000 00:00:08.6
        76 16-JUL-14 10.55.03.421 AM                          +00000 00:00:01.7
        77 16-JUL-14 11.02.59.311 AM                          +00000 00:00:02.8
        78 16-JUL-14 11.15.32.171 AM                          +00000 00:00:02.0
        79 16-JUL-14 11.30.36.014 AM                          +00000 00:00:01.2
        80 16-JUL-14 11.45.39.828 AM                          +00000 00:00:01.7

6 rows selected.
SYS@linora> BEGIN
  2    DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE(low_snap_id  => 77,
  3                                                 high_snap_id => 80,
  4                                                 dbid         => 3385851293);
  5  END;
  6  /
PL/SQL procedure successfully completed.
SYS@linora> select SNAP_ID,BEGIN_INTERVAL_TIME,FLUSH_ELAPSED from dba_hist_snapshot 
where snap_id>=75 and snap_id <=80;
   SNAP_ID BEGIN_INTERVAL_TIME                                FLUSH_ELAPSED
---------- -------------------------------------------------- ------------------------------
        75 16-JUL-14 10.50.34.000 AM                          +00000 00:00:08.6
        76 16-JUL-14 10.55.03.421 AM                          +00000 00:00:01.7
```
需要注意的是，调用<code>DROP_SNAPSHOT_RANGE</code>删除snapshot时，在此期间内的ASH也会被删除。
<h3>2.管理基线</h3>
Baseline包含指定时间点的性能数据，一般是在业务峰值性能正常时的数据，因此，当某天性能出现异常时，则可与baseline进行比较。基线有几种类型：单一基线，重复基线，及11G的Moving Window Baseline，这里只涉及最简单的单一基线。
##### 创建baseline
baseline是在两个snapshot之间的，因此，首先从<code>dba_hist_snapshot</code>决定baseline创建在哪两个snapshot之间，然后，使用<code>CREATE_BASELINE</code>进行创建。
```
SYS@linora> BEGIN
  2    DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE(start_snap_id => 81,
  3                                             end_snap_id   => 85,
  4                                             baseline_name => 'test baseline',
  5                                             dbid          => 3385851293,
  6                                             expiration    => 30);
  7  END;
  8  /

PL/SQL procedure successfully completed.
```
该procedure最后一个参数expiration，在本例中表示30天后该baseline过期，如果不指定此参数，则表示永不过期。
##### 查看baseline
```
SYS@linora> select baseline_name, baseline_type, start_snap_id, end_snap_id, expiration
  2    from DBA_HIST_BASELINE;

BASELINE_NAME                            BASELINE_TYPE START_SNAP_ID END_SNAP_ID EXPIRATION
---------------------------------------- ------------- ------------- ----------- ----------
test baseline                            STATIC                   81          85         30
SYSTEM_MOVING_WINDOW                     MOVING_WINDOW            75         104
```
这里有个小提示，针对DBA或者其他相关视图，如果忘记此视图叫什么名字，可利用dict数据字典进行模糊查找。
```
select * from dict where table_name like 'DBA_HIST%';
```
##### 删除baseline
```
SYS@linora> BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_BASELINE(baseline_name => 'test baseline',
                                         cascade       => FALSE,
                                         dbid          => 3385851293);
END;
/
```
如果cascade为false，则表示仅仅删除baseline，如果cascade参数为true，则表示相关快照也一起删除。
<h3>3.生成AWR报告</h3>
一般默认执行@?/rdbms/admin/awrrpt.sql，如果需要生成RAC的报告，则执行@?/rdbms/admin/awrgrpt.sql，如果RAC上需要收集其他节点的AWR报告，则执行@?/rdbms/admin/awrrpti.sql。11g同时新增了一个awrddrpt.sql，这个脚本用来比较两个时间段的AWR报告(RAC中使用的是awrgdrpt.sql)。
##### 使用DBMS_WORKLOAD_REPOSITORY手动执行awr报告
Function AWR_REPORT_HTML定义如下：
```
DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
   l_dbid       IN    NUMBER,
   l_inst_num   IN    NUMBER,
   l_bid        IN    NUMBER,
   l_eid        IN    NUMBER,
   l_options    IN    NUMBER DEFAULT 0)
 RETURN awrrpt_text_type_table PIPELINED;
```
参数解析：l_dbid表示数据库dbid，l_inst_num表示实例号，l_bid表示开始的snapshot id，l_eid表示结束的snapshot id，l_options控制报表的输出，只有一个值：8，则表示输出中带ADDM的建议值。
```
SYS@linora> spool testawr.html
SYS@linora> select output from table (DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(3385851293,1,75,76,8));
SYS@linora> spool off
```

Reference:  
[Oracle® Database Performance Tuning Guide 11g Release 2 (11.2)](http://docs.oracle.com/cd/E11882_01/server.112/e41573/toc.htm)  
[Oracle® Database PL/SQL Packages and Types Reference 11g Release 2 (11.2)](http://docs.oracle.com/cd/E11882_01/appdev.112/e40758/d_workload_repos.htm#ARPLS093)
