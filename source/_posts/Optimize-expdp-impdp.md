---
title: '优化expdp/impdp思路'
categories: oracle
comments: false
date: 2019-09-16 11:16:33
tags:
---

## expdp

1. RAC环境中，设置cluster=n参数
2. 排除统计信息导出: exclude=statistics
3. 并行导出: parallel=Number Of CPUs
4. RAC环境中，设置PARALLEL_FORCE_LOCAL=TRUE
5. 12c中，添加version=12.1以增强兼容性

示例：
```sql
expdp system/manager@xxxx schemas=xxx \
cluster=n exclude=statistics parallel=4 \
PARALLEL_FORCE_LOCAL=true version=12.1 \
directory=xxx dumpfile=xxx.dmp
```


## impdp
1. 导入数据后再建立索引、约束等, 以避免产生大量的undo和temp

```sql
--生成DDl语句
impdp system/manager@xxxx directory=xxx dumpfile=xxx.dmp \
sqlfile=xxx.sql include=constraints, indexes
```
根据sql语句保留约束、索引的创建语句再进行导入：
```sql
impdp system/xxxxxxxx directory=datapump dumpfile=xxx.dump \
EXCLUDE=STATISTICS,constraints,indexes \
remap_tablespace=aaaa:bbbb parallel=4 \
remap_schema=xxx:xxx transform=disable_archive_logging:Y
```

2. 尽量不要使用TABLE_EXISTS_ACTION=APPEND or TABLE_EXISTS_ACTION=TRUNCATE
3. 导入时添加transform=disable_archive_logging:Y参数，12c新特性，可以在导入的时候减少redo的产生


Reference:
[Error ORA-30036 DataPump Import (IMPDP) Exhausts Undo Tablespace (Doc ID 727894.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=171869790919870&parent=DOCUMENT&sourceId=735366.1&id=727894.1&_afrWindowMode=0&_adf.ctrl-state=1ns466yfl_109)
[Import DataPump - How To Limit The Amount Of UNDO Generation of an IMPDP job ? (Doc ID 1670349.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=171930816415232&parent=DOCUMENT&sourceId=727894.1&id=1670349.1&_afrWindowMode=0&_adf.ctrl-state=1ns466yfl_158)



__EOF__
