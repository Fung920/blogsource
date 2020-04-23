---
title: '等待事件: reliable message'
categories: oracle
comments: false
date: 2020-04-23 10:42:40
tags: 
 - tuning
 - wait interface
---

开发人员反馈，最近批量作业相当慢，原来只需要10-20分钟，现在已经快到2小时。
从AWR看，DB time不算高，但是有几个等待事件比较明显：
![top event](/images/reliable_message1.jpg)

以下为相同时间段的`gv$channel_waits`:
```sql
SQL> select channel, sum(wait_count) sum_wait_count from gv$channel_waits group by channel order by 2 desc;

CHANNEL                                                          SUM_WAIT_COUNT
---------------------------------------------------------------- --------------
kxfp control signal channel                                            17159048
Result Cache: Channel                                                  13350997
MMON remote action broadcast channel                                     552948
obj broadcast channel                                                    398453
RBR channel                                                              126023
kxfp remote slave spawn channel                                           11439
parameters to cluster db instances - broadcast channel                     2603
kill job broadcast - broadcast channel                                     2602
service operations - broadcast channel                                     2586
GEN0 ksbxic channel                                                        1699
CKPT ksbxic channel                                                        1333
LCK0 ksbxic channel                                                         960
quiesce channel                                                               2
```

从以上视图看到，"kxfp control signal channel"跟"Result Cache: Channel"是等待比较严重的channel。

## 1. "kxfp control signal channel"
该channel跟_BUG 20470877:LONG WAITS FOR "RELIABLE MESSAGE" AFTER A FEW DAYS OF UPTIME_有关,同时在后台等待时间中，"CSS group membership query"等待较高，可以判定跟这个bug有很大关系。
![CSS group membership query](/images/reliable_message3.jpg)

在停机时间安装完补丁后，开发同事反馈批量处理时间恢复正常,同时,AWR报告中，相关等待事件大大减少或者消失。
![after patching](/images/reliable_message2.jpg)

## 2. Result Cache: Channel
该等待事件跟结果集缓存相关，建议关闭结果集缓存:
```sql
alter system set result_cache_max_size=0 scope=both sid='*';
```

最近在处理性能问题时，"latch free"出现频率很高，且均与"Result Cache"相关，在新建系统中，12cR1建议关闭此功能。


__EOF__
