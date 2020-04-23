---
title: Oracle 12c开启结果集缓存导致大量latch free等待
categories: oracle
comments: false
date: 2020-04-20 15:47:38
tags: 
- latch free
- result cache
---
最近在业务监控上，发现某个更新模块自4月9日起延时较高，从AWR看，`latch free`等待特别高，且平均等待时间也比较长。
![latch free](/images/latch_free.jpg)
从ADDM查看，该latch ID为559，`Result Cache: RC Latch`，

![addm report](/images/latch_free_addm.jpg)

该库是开启了结果集缓存：
```sql
SQL> show parameter result

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
client_result_cache_lag              big integer 3000
client_result_cache_size             big integer 0
result_cache_max_result              integer     5
result_cache_max_size                big integer 198592K
result_cache_mode                    string      MANUAL
result_cache_remote_expiration       integer     0
```
在AWR Top sql中，频繁出现：
```sql
SELECT /* DS_SVC */ /*+ dynamic_sampling(0) no_sql_tune no_monitoring optimizer_features_enable(default) no_parallel result_cache(snapshot=3600) OPT_ESTIMATE(@"innerQuery", TABLE, "XX", ROWS=30319104.08) */ C1, C2, C3 FROM (SELECT /*+
```

__原因__:
在12.1.0.2版本中，默认启用了Adaptive Dynamic Statistics，自动决定动态统计信息是否是有用的和在所有的 SQL 语句上使用哪个统计级别。当优化器认为有必要的时候，它就会收集动态统计信息。
当SQL语句使用了Adaptive Dynamic Statistics (Sampling)，那么基于它的统计信息，系统认为对这些语句使用result cache可以提升性能。这可能会导致result cache的大量使用并引发"Result Cache: RC Latch"争用。

Workaround:
* 1. 关闭自动动态统计信息使用结果缓存的机制

```sql
alter system set "_optimizer_ads_use_result_cache" = FALSE sid='*' scope=both;
```

* 2. 关闭结果集缓存

```sql
alter system set result_cache_max_size=0 scope=both;
```

由于业务无法停机，为了避免在线业务正在使用结果集而导致业务失败，当前采取关闭ADS，待停机时间再关闭结果集缓存.


_相关初始化参数_:

* RESULT_CACHE_MODE : 该参数有两个值
    - Manual: 手动模式下，需要使用"result_cache" hint才能启用结果集缓存
    - Force: 所有查询结果都会使用到结果集缓存

* `_optimizer_ads_use_result_cache` , 如果是true(默认), 数据库自动收集动态采样信息，并使用结果集缓存这些采样结果




Reference:
[当在 Oracle 12c 上设置 RESULT_CACHE_MODE = MANUAL 时发生'Result Cache: RC Latch'类型的”Latch Free”等待 (Doc ID 2102499.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=98181864050001&id=2102499.1&_afrWindowMode=0&_adf.ctrl-state=ix3ibxic_110)
[High Waits on ‘enq: RC – Result Cache: Contention’ when Results are Updated Frequently (Doc ID 1623225.1)]()


__EOF__
