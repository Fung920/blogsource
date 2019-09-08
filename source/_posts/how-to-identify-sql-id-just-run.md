---
title: 如何确认刚刚执行的SQL ID
categories: oracle
comments: false
date: 2019-09-04 18:26:53
tags: ['how to','sql_id']
---
在一些特定场合，需要对刚刚执行的sql进行一些监控或者优化，因此需要找出SQL的sql_id，经常会用v$sql中的sql_text去关联查找。
下面是另一种方法可以确定这个SQL ID。

在`v$session`中，__PREV_SQL_ID__就是表示上一次执行的SQL ID。但如果直接去查找，你会发现，不管你执行什么SQL，这个值都是9babjv8yq8ru3。
```sql
select sql_text from v$sqltext
where sql_id = '9babjv8yq8ru3';

SQL_TEXT
----------------------------------------------------------------
BEGIN DBMS_OUTPUT.GET_LINES(:LINES, :NUMLINES); END;
```

这是因为默认配置下, serveroutput设置为ON；此时在后台数据库会将dbms_output的缓存内容打印到屏幕，这个语句就是调用这个功能。
因此，只需要把serveroutput设置为OFF，我们的目的就达到了。

```sql
set serveroutput off
--run sql here
select prev_sql_id from v$session where sid=sys_context('userenv','sid');
```


__EOF__
