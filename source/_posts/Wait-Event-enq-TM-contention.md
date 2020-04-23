---
title: '等待事件enq:TM contention'
categories: oracle
comments: false
date: 2019-09-16 09:19:11
tags:
- tuning
- wait interface
---
开发人员反应在进行大量insert操作时，速度很慢，平时只要几分钟，但目前跑了几个小时仍旧没结束。

诊断步骤：

* 从活动会话查找相关信息

```sql
set linesize 200
set pagesize 100
clear columns
col inst for 99999999
col sid for 9990
col serial# for 999990
col username for a12
col osuser for a16
col program for a10 trunc
col Locked for a6
col status for a1 trunc print
col "hh:mm:ss" for a8
col SQL_ID for a15
col seq# for 99990
col event heading 'Current/LastEvent' for a25 trunc col state head 'State (sec)' for a14
select inst_id inst, sid , serial# , username
, ltrim(substr(osuser, greatest(instr(osuser, '\', -1, 1)+1,length(osuser)-14))) osuser , substr(program,instr(program,'/',-
1)+1,decode(instr(program,'@'),0,decode(instr(program,'.'),0,length(program),instr(program,'.')- 1),instr(program,'@')-1)) program, decode(lockwait,NULL,' ','L') locked, status, to_char(to_date(mod(last_call_et,86400), 'sssss'), 'hh24:mi:ss') "hh:mm:ss"
, SQL_ID, seq# , event,
decode(state,'WAITING','WAITING '||lpad(to_char(mod(SECONDS_IN_WAIT,86400),'99990'),6)
,'WAITED SHORT TIME','ON CPU','WAITED KNOWN TIME','ON CPU',state) state , substr(module,1,25) module, substr(action,1,20) action
from GV$SESSION
where type = 'USER'
and audsid != 0    -- to exclude internal processess
order by inst_id, status, last_call_et desc, sid
/
```

* 从等待事件查看相关信息

```sql
SELECT NVL(a.event, 'ON CPU') AS event,
       COUNT(*) AS total_wait_time
FROM   gv$active_session_history a
WHERE  a.sample_time > SYSDATE - 5/(24*60) -- 5 mins
GROUP BY a.event
ORDER BY total_wait_time DESC;
```

通过以上sql定位到用户SQL相关信息，其等待事件为`enq: TM-contention`。
对于这个等待事件有以下几种可能：

* 有外键约束的子表缺少索引
* 并行直接路径插入
* Analyze Index Validate Structure
* 使用insert append导致


通过SQL ID获取的SQL TEXT发现，该SQL使用了insert append，修改SQL取消append，顺利插入。

附，其他思路:

```sql
--通过gv$active_history_session/dba_active_sess_history查找相关SQL
set line 300 pagesize 300
col event for a30
col machine for a20
col program for a20
select sql_id,event,machine,program,p1,p2,p3,
blocking_session
--from DBA_HIST_ACTIVE_SESS_HISTORY
from gv$active_session_history
where lower(event) like '%enq%tm%'
and sample_time >= to_date('2019-09-16 08:26','yyyy-mm-dd hh24:mi')
and sample_time < to_date('2019-09-16 09:26','yyyy-mm-dd hh24:mi');
```

```sql
--查找对应排名第一的sql id
select sql_id,count(1)from v$active_session_history
where lower(event) like '%enq%tm%'
and sample_time >= to_date('2019-09-16 08:26','yyyy-mm-dd hh24:mi')
and sample_time < to_date('2019-09-16 09:26','yyyy-mm-dd hh24:mi')
group by sql_id;
```

如果是缺失外键索引，则添加对应的外键索引即可。
```sql
--查找缺失索引的外键
SELECT * FROM (
SELECT c.table_name, cc.column_name, cc.position column_position
FROM   user_constraints c, user_cons_columns cc
WHERE  c.constraint_name = cc.constraint_name
AND    c.constraint_type = 'R'
MINUS
SELECT i.table_name, ic.column_name, ic.column_position
FROM   user_indexes i, user_ind_columns ic
WHERE  i.index_name = ic.index_name
)
ORDER BY table_name, column_position;
```



Reference:
[WAITEVENT: "enq: TM - contention" Reference Note (Doc ID 1980175.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=167068159961386&id=1980175.1&_adf.ctrl-state=etnj58m6_99)
[High Enq: TM - Contention Wait Events When Using Insert APPEND (Doc ID 2247733.1) ](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=167064474681406&id=2247733.1&_afrWindowMode=0&_adf.ctrl-state=etnj58m6_4)
[Resolving Issues Where 'enq: TM - contention' Waits are Occurring (Doc ID 1905174.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=167065737904370&id=1905174.1&_afrWindowMode=0&_adf.ctrl-state=etnj58m6_45)
__EOF__
