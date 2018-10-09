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
see mos: [Difference in Tablespace Size Values From dba_data_files and dba_tablespace_usage_metrics/V$filespace_usage (Doc ID 455715.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=239212711068322&id=455715.1&_adf.ctrl-state=uz8pga6j_119)

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
   >see mos: [Mismatch Between V$TEMP_SPACE_HEADER and V$TEMPSEG_USAGE/V$SORT_USAGE (Doc ID 2095211.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=240558702386283&id=2095211.1&_adf.ctrl-state=uz8pga6j_527).

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


__EOF__
