---
title: Correct way to monitor tablespace usages via SQLs
categories: oracle
comments: false
date: 2018-10-09 07:26:05
tags: scripts
---
There are many ways to monitor Oracle tablespace usages. OEM, SQL Developer, ect,. But in some cases, we have no handy GUI tools, which require us to use SQLs for monitoring tablespace usages.
Some v$ views and dictionaries are useful while monitoring tablespace.
<!--more-->

# 1. Permanent tablespace
* `DBA_TABLESPACE_USAGE_METRICS`
   Describes tablespace usage metrics for all types of tablespaces, including permanent, temporary, and undo tablespaces. Not suitable for autoextend data files.
   > TABLESPACE_SIZE is the maximum possible size if AUTO extended on, not the current size. The same applies to USED_PERCENT.
   > USED_SPACE and TABLESPACE_SIZE are in blocks.

see MOS: [Difference in Tablespace Size Values From dba_data_files and dba_tablespace_usage_metrics/V$filespace_usage (Doc ID 455715.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=239212711068322&id=455715.1&_adf.ctrl-state=uz8pga6j_119)

For the usage of tablespace(must exclude autoextensible data files):
```sql
set line 200
col tablespace_name for a30
SELECT a.tablespace_name,
  ROUND((a.used_space * b.block_size)/1024/1024, 2) AS "USED_SPACE(MB)",
  ROUND((a.tablespace_size * b.block_size)/1024/1024, 2) AS "TABLESPACE_SIZE(MB)",
  ROUND(a.used_percent, 2) AS "USED_PERCENT"
  FROM DBA_TABLESPACE_USAGE_METRICS a JOIN
  DBA_TABLESPACES b
  ON a.tablespace_name = b.tablespace_name;
ABLESPACE_NAME            USED_SPACE(MB) TABLESPACE_SIZE(MB) USED_PERCENT
------------------------------ -------------- ------------------- ------------
SYSTEM                       828.13          4741.98     17.46    --autoextend on, not real
SYSAUX                       645.94          4741.98     13.62    --autoextend on, not real
TEMP                              3          4741.98       .06
USERS                        7920.25         8322.5      95.17    --auto extend off, real usages
```

* `v$filespace_usage`
   Summarizes space allocation information of each datafile and tempfile. This view is the basic view of `DBA_TABLESPACE_USAGE_METRICS`.
* `DBA_HIST_TBSPC_SPACE_USAGE`
   Displays historical tablespace usage statistics, deponds on AWR retention

Below shows the examples:
## 1.1 Monitoring real tablespace usage
```sql
col dbname for a10
col tbs_name for a30
col sum_size for a30
col free_size for a30
col usage_pct for a30
SELECT * FROM (
    SELECT  substr (d.NAME || '                                                ', 1, 15) AS DBNAME,
    SUBSTR (a.tablespace_name || '                                     ', 1, 30) AS TBS_NAME,
    SUBSTR (a.BYTES ||'                                                ', 1, 8) AS SUM_SIZE,
    SUBSTR (TO_CHAR(decode(ROUND(c.BYTES,1),null,0,ROUND(c.bytes,1)))||'              ', 1, 8) AS FREE_SIZE,
    substr (ROUND (100 * (1 - (NVL (c.BYTES, 0)
                    / NVL (a.BYTES, 0))), 2)||'   ',1,5) AS USAGE_PCT
    FROM (SELECT tablespace_name,SUM (BYTES) / 1024 / 1024 AS BYTES
        FROM dba_data_files
        GROUP BY tablespace_name) a,
    (SELECT f.tablespace_name,SUM (f.BYTES) / 1024 / 1024 AS BYTES
        FROM dba_free_space f
        GROUP BY f.tablespace_name) c,
    v$database d
    WHERE a.tablespace_name = c.tablespace_name(+)
);
```

## 1.2 Monitoring tablespace growth rate
```sql
SELECT A.NAME, B.TABLESPACE_ID,B.DATETIME,B.USED_SIZE_MB,B.INC_MB,
       CASE WHEN SUBSTR(INC_RATE,1,1)='.' THEN '0'||INC_RATE
            WHEN SUBSTR(INC_RATE,1,2)='-.' THEN '-0'||SUBSTR(INC_RATE,2,LENGTH(INC_RATE))
            ELSE INC_RATE
       END AS INC_RATEX
  FROM V$TABLESPACE A,
       (
           SELECT TABLESPACE_ID,DATETIME,
                  USED_SIZE_MB,
                  (DECODE(PREV_USE_MB,0,0,USED_SIZE_MB)-PREV_USE_MB) AS  INC_MB,
                  TO_CHAR(ROUND((DECODE(PREV_USE_MB,0,0,USED_SIZE_MB)-PREV_USE_MB)/DECODE(PREV_USE_MB,0,1,PREV_USE_MB)*100,2))||'%' AS INC_RATE
         FROM
         (
           SELECT TABLESPACE_ID,
                  TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss')) DATETIME,
                  MAX(TABLESPACE_USEDSIZE * 8 / 1024) USED_SIZE_MB,
                  LAG(MAX(TABLESPACE_USEDSIZE * 8 / 1024),1,0) OVER(PARTITION BY TABLESPACE_ID ORDER BY TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss')) ) AS PREV_USE_MB
             FROM DBA_HIST_TBSPC_SPACE_USAGE
            WHERE TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss')) > TRUNC(SYSDATE - 30)
            GROUP BY TABLESPACE_ID, TRUNC(TO_DATE(RTIME, 'mm/dd/yyyy hh24:mi:ss'))
         )
       ) B
 WHERE A.TS# = B.TABLESPACE_ID
 ORDER BY B.TABLESPACE_ID,DATETIME;
```

# 2. Temporary tablespace
* `GV_$TEMP_SPACE_HEADER`
   The views `v$sort_usage` or `v$tempseg_usage` ( and `v$sort_segment`) give the correct information regarding the allocation of sort segments. The view `v$temp_space_header` shows that these many blocks were touched in each temp file at some point when temp usage was at its highest,in essence, it shows the number of initialized blocks for each tempfile, not the actual allocated blocks.

See MOS: [Mismatch Between V$TEMP_SPACE_HEADER and V$TEMPSEG_USAGE/V$SORT_USAGE (Doc ID 2095211.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=240558702386283&id=2095211.1&_adf.ctrl-state=uz8pga6j_527).

* Correct way to show TEMP tablespace usage
```sql
select tablespace_name,
tablespace_size/1024/1024 "Total Space",
allocated_space/1024/1024 "Alloc Space",
free_space/1024/1024 "Free Space"
from dba_temp_free_space;

select tablespace_name, total_blocks*8/1024 total_mb, used_blocks*8/1024 used_mb, free_blocks*8/1024 free_mb
from v$sort_segment;

SELECT D.tablespace_name,
       SPACE "SUM_SPACE(M)",
       blocks "SUM_BLOCKS",
       used_space "USED_SPACE(M)",
       Round(Nvl(used_space, 0) / SPACE * 100, 2) "USED_RATE(%)",
       SPACE - used_space "FREE_SPACE(M)"
  FROM (SELECT tablespace_name,
               Round(SUM(bytes) / (1024 * 1024), 2) SPACE,
               SUM(blocks) BLOCKS
          FROM dba_temp_files
         GROUP BY tablespace_name) D,
       (SELECT tablespace,
               Round(SUM(blocks * 8192) / (1024 * 1024), 2) USED_SPACE
          FROM v$sort_usage
         GROUP BY tablespace) F
 WHERE D.tablespace_name = F.tablespace(+)
```

# 3. Get table/segment growth history
MOS: [How To Get Table Growth History Information? (Doc ID 1395195.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=320948395693558&parent=EXTERNAL_SEARCH&sourceId=HOWTO&id=1395195.1&_afrWindowMode=0&_adf.ctrl-state=tc5cvv4es_4)

* `DBA_HIST_SEG_STAT`
    DBA_HIST_SEG_STAT displays historical information about segment-level statistics. This view captures the top segments based on a set of criteria and captures information from V$SEGSTAT.

View the object (segment) growth in blocks:
```sql
column owner format a16
column object_name format a36
column start_day format a11
column block_increase format 9999999999

select   obj.owner, obj.object_name,
         to_char(sn.BEGIN_INTERVAL_TIME,'RRRR-MON-DD') start_day,
         sum(a.SPACE_USED_DELTA) block_increase_bytes
from     dba_hist_seg_stat a,
         dba_hist_snapshot sn,
         dba_objects obj
where    sn.snap_id = a.snap_id
and      obj.object_id = a.obj#
and      obj.owner not in ('SYS','SYSTEM')
and      end_interval_time between to_timestamp('01-JAN-2012','DD-MON-RRRR')
         and to_timestamp('02-FEB-2012','DD-MON-RRRR')
group by obj.owner, obj.object_name,
         to_char(sn.BEGIN_INTERVAL_TIME,'RRRR-MON-DD')
order by obj.owner, obj.object_name
/
```

Segments with highest growth (Top n):
```sql
SELECT o.OWNER , o.OBJECT_NAME , o.SUBOBJECT_NAME , o.OBJECT_TYPE ,
    t.NAME "Tablespace Name", s.growth/(1024*1024) "Growth in MB",
    (SELECT sum(bytes)/(1024*1024)
    FROM dba_segments
    WHERE segment_name=o.object_name) "Total Size(MB)"
FROM DBA_OBJECTS o,
    ( SELECT TS#,OBJ#,
        SUM(SPACE_USED_DELTA) growth
    FROM DBA_HIST_SEG_STAT
    GROUP BY TS#,OBJ#
    HAVING SUM(SPACE_USED_DELTA) > 0
    ORDER BY 2 DESC ) s,
    v$tablespace t
WHERE s.OBJ# = o.OBJECT_ID
AND s.TS#=t.TS#
AND rownum < 51
ORDER BY 6 DESC
/
```

Script to display table size changes between two periods:
```sql
column "Percent of Total Disk Usage" justify right format 999.99
column "Space Used (MB)" justify right format 9,999,999.99
column "Total Object Size (MB)" justify right format 9,999,999.99
set linesize 150
set pages 80
set feedback off
select * from (select to_char(end_interval_time, 'MM/DD/YY') mydate,
sum(space_used_delta) / 1024 / 1024 "Space used (MB)",
avg(c.bytes) / 1024 / 1024 "Total Object Size (MB)",
round(sum(space_used_delta) / sum(c.bytes) * 100, 2) "Percent of Total Disk Usage"
from
   dba_hist_snapshot sn,
   dba_hist_seg_stat a,
   dba_objects b,
   dba_segments c
where begin_interval_time > trunc(sysdate) - &days_back
and sn.snap_id = a.snap_id
and b.object_id = a.obj#
and b.owner = c.owner
and b.object_name = c.segment_name
and c.segment_name = '&segment_name'
group by to_char(end_interval_time, 'MM/DD/YY'))
order by to_date(mydate, 'MM/DD/YY');
```

With above information, it's easier to ask application owner if the growth is normal or not.

__EOF__
