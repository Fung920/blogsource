---
layout: post
title: "如何获得SQL的执行计划"
date: 2014-08-20 09:52:36
comments: false
categories: oracle
tags: tuning
keywords: Oracle执行计划
description: 怎么查看SQL的执行计划
---
Oracle中的执行计划显示在执行一条SQL语句时必须执行的详细步骤，通常以表格形式呈现，但其实是树形结构。查看Oracle中的执行计划一般有以下几种方法(包括但不限于)。
<!--more-->
### 1. explain plan
<code>explain plan</code>只显示一条SQL的执行计划，但不会真正去执行它。当使用此命令生成执行计划后，还需要调用<code>dbms_xplan</code>包去查看相关内容。在TOAD或者PL/SQL Developer查看的执行计划，其实也就是explain plan的变体，因此，有可能是不准确的执行计划。
```
SH@linora> explain plan for 
  2  SELECT /*+ gather_plan_statistics */ --显示统计信息，相当于开启参数statistics_level=all;
  3  p.prod_name as product, sum(s.quantity_sold) as units 
  4  FROM sales s, products p
  5  WHERE s.prod_id =p.prod_id
  6  GROUP BY p.prod_name;
Explained.
SH@linora> select * from table(dbms_xplan.display());
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------
Plan hash value: 504757596
----------------------------------------------------------------------------------------------------
| Id  | Operation               | Name     | Rows  | Bytes | Cost (%CPU)| Time     | Pstart| Pstop |
----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |          |    71 |  3337 |   589  (12)| 00:00:08 |       |       |
|   1 |  HASH GROUP BY          |          |    71 |  3337 |   589  (12)| 00:00:08 |       |       |
|*  2 |   HASH JOIN             |          |    72 |  3384 |   588  (12)| 00:00:08 |       |       |
|   3 |    VIEW                 | VW_GBC_5 |    72 |  1224 |   585  (12)| 00:00:08 |       |       |
|   4 |     HASH GROUP BY       |          |    72 |   504 |   585  (12)| 00:00:08 |       |       |
|   5 |      PARTITION RANGE ALL|          |   918K|  6281K|   533   (3)| 00:00:07 |     1 |    28 |
|   6 |       TABLE ACCESS FULL | SALES    |   918K|  6281K|   533   (3)| 00:00:07 |     1 |    28 |
|   7 |    TABLE ACCESS FULL    | PRODUCTS |    72 |  2160 |     3   (0)| 00:00:01 |       |       |
----------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("ITEM_1"="P"."PROD_ID")
19 rows selected.
```
<code>explain plan</code>其实是将Oracle所产生的执行计划步骤写入PLAN_TABLE$，此表是一个全局临时表，因此，各个session只能看到自己执行的SQL所产生的执行计划。
```
SYS@linora> set long 9999
SYS@linora> set pagesize 9999
SYS@linora> select dbms_metadata.get_ddl('TABLE','PLAN_TABLE$','SYS') from dual;
DBMS_METADATA.GET_DDL('TABLE','PLAN_TABLE$','SYS')
--------------------------------------------------------------------------------
  CREATE GLOBAL TEMPORARY TABLE "SYS"."PLAN_TABLE$"
   (    "STATEMENT_ID" VARCHAR2(30),
        "PLAN_ID" NUMBER,
        "TIMESTAMP" DATE,
        "REMARKS" VARCHAR2(4000),
        "OPERATION" VARCHAR2(30),
        "OPTIONS" VARCHAR2(255),
        "OBJECT_NODE" VARCHAR2(128),
        "OBJECT_OWNER" VARCHAR2(30),
        "OBJECT_NAME" VARCHAR2(30),
        "OBJECT_ALIAS" VARCHAR2(65),
        "OBJECT_INSTANCE" NUMBER(*,0),
        "OBJECT_TYPE" VARCHAR2(30),
        "OPTIMIZER" VARCHAR2(255),
        "SEARCH_COLUMNS" NUMBER,
        "ID" NUMBER(*,0),
        "PARENT_ID" NUMBER(*,0),
        "DEPTH" NUMBER(*,0),
        "POSITION" NUMBER(*,0),
        "COST" NUMBER(*,0),
        "CARDINALITY" NUMBER(*,0),
        "BYTES" NUMBER(*,0),
        "OTHER_TAG" VARCHAR2(255),
        "PARTITION_START" VARCHAR2(255),
        "PARTITION_STOP" VARCHAR2(255),
        "PARTITION_ID" NUMBER(*,0),
        "OTHER" LONG,
        "OTHER_XML" CLOB,
        "DISTRIBUTION" VARCHAR2(30),
        "CPU_COST" NUMBER(*,0),
        "IO_COST" NUMBER(*,0),
        "TEMP_SPACE" NUMBER(*,0),
        "ACCESS_PREDICATES" VARCHAR2(4000),
        "FILTER_PREDICATES" VARCHAR2(4000),
        "PROJECTION" VARCHAR2(4000),
        "TIME" NUMBER(*,0),
        "QBLOCK_NAME" VARCHAR2(30)
   ) ON COMMIT PRESERVE ROWS
```
### 2. dbms_xplan
dbms_xplan有好几种调用方法，以下仅列出常用的三种方法(后面两种适合数据库在10g及以上的版本)：
<li>display--输出plan table内容</li>
<li>display_cursor--用于显示内存中的SQL执行计划</li>
<li>display_awr--输出AWR中的历史SQL执行计划</li>
#### 2.1 DISPLAY
语法：
```
DBMS_XPLAN.DISPLAY(
   table_name    IN  VARCHAR2  DEFAULT 'PLAN_TABLE',
   statement_id  IN  VARCHAR2  DEFAULT  NULL, 
   format        IN  VARCHAR2  DEFAULT  'TYPICAL',
   filter_preds  IN  VARCHAR2 DEFAULT NULL);
```
不加任何参数：
```
SCOTT@linora> EXPLAIN PLAN FOR
  2  SELECT * FROM emp e, dept d
  3     WHERE e.deptno = d.deptno
  4     AND e.ename='benoit';
Explained.
SCOTT@linora> SET LINESIZE 130
SCOTT@linora> SET PAGESIZE 0
SCOTT@linora> SELECT * FROM table(DBMS_XPLAN.DISPLAY);
Plan hash value: 3625962092
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |     1 |    58 |     4   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |         |     1 |    58 |     4   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |         |     1 |    58 |     4   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | EMP     |     1 |    38 |     3   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PK_DEPT |     1 |       |     0   (0)| 00:00:01 |
|   5 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     1 |    20 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   3 - filter("E"."ENAME"='benoit')
   4 - access("E"."DEPTNO"="D"."DEPTNO")
18 rows selected.
```
添加ADVANCED参数(显示所有信息)：
```
SCOTT@linora>   SELECT * FROM table(DBMS_XPLAN.DISPLAY('','','ADVANCED'));
Plan hash value: 3625962092
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |     1 |    58 |     4   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |         |     1 |    58 |     4   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |         |     1 |    58 |     4   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | EMP     |     1 |    38 |     3   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PK_DEPT |     1 |       |     0   (0)| 00:00:01 |
|   5 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     1 |    20 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
   1 - SEL$1
   3 - SEL$1 / E@SEL$1
   4 - SEL$1 / D@SEL$1
   5 - SEL$1 / D@SEL$1
Outline Data
-------------
  /*+
      BEGIN_OUTLINE_DATA
      NLJ_BATCHING(@"SEL$1" "D"@"SEL$1")
      USE_NL(@"SEL$1" "D"@"SEL$1")
      LEADING(@"SEL$1" "E"@"SEL$1" "D"@"SEL$1")
      INDEX(@"SEL$1" "D"@"SEL$1" ("DEPT"."DEPTNO"))
      FULL(@"SEL$1" "E"@"SEL$1")
      OUTLINE_LEAF(@"SEL$1")
      ALL_ROWS
      DB_VERSION('11.2.0.4')
      OPTIMIZER_FEATURES_ENABLE('11.2.0.4')
      IGNORE_OPTIM_EMBEDDED_HINTS
      END_OUTLINE_DATA
  */
Predicate Information (identified by operation id):
---------------------------------------------------
   3 - filter("E"."ENAME"='benoit')
   4 - access("E"."DEPTNO"="D"."DEPTNO")
Column Projection Information (identified by operation id):
-----------------------------------------------------------
   1 - (#keys=0) "E"."EMPNO"[NUMBER,22], "E"."ENAME"[VARCHAR2,10],
       "E"."JOB"[VARCHAR2,9], "E"."MGR"[NUMBER,22], "E"."HIREDATE"[DATE,7],
       "E"."SAL"[NUMBER,22], "E"."COMM"[NUMBER,22], "E"."DEPTNO"[NUMBER,22],
       "D"."DEPTNO"[NUMBER,22], "D"."DNAME"[VARCHAR2,14], "D"."LOC"[VARCHAR2,13]
   2 - (#keys=0) "E"."EMPNO"[NUMBER,22], "E"."ENAME"[VARCHAR2,10],
       "E"."JOB"[VARCHAR2,9], "E"."MGR"[NUMBER,22], "E"."HIREDATE"[DATE,7],
       "E"."SAL"[NUMBER,22], "E"."COMM"[NUMBER,22], "E"."DEPTNO"[NUMBER,22],
       "D".ROWID[ROWID,10], "D"."DEPTNO"[NUMBER,22]
   3 - "E"."EMPNO"[NUMBER,22], "E"."ENAME"[VARCHAR2,10], "E"."JOB"[VARCHAR2,9],
       "E"."MGR"[NUMBER,22], "E"."HIREDATE"[DATE,7], "E"."SAL"[NUMBER,22],
       "E"."COMM"[NUMBER,22], "E"."DEPTNO"[NUMBER,22]
   4 - "D".ROWID[ROWID,10], "D"."DEPTNO"[NUMBER,22]
   5 - "D"."DNAME"[VARCHAR2,14], "D"."LOC"[VARCHAR2,13]
61 rows selected.
```
DISPLAY仅仅针对预估的执行计划，而不是真实的执行计划，尤其当SQL语句包含绑定变量时。
#### 2.2 DISPLAY_CURSOR
语法：
```
DBMS_XPLAN.DISPLAY_CURSOR(
   sql_id        IN  VARCHAR2  DEFAULT  NULL,
   child_number  IN  NUMBER    DEFAULT  NULL, 
   format        IN  VARCHAR2  DEFAULT  'TYPICAL');
```
DISPLAY_CURSOR是显示内存中的执行计划，只要目标SQL的执行计划所在的child cursor还在shared pool中，就可以使用display_cursor来查看：
```
SCOTT@linora> SELECT /*+ display_cursor_example */ * FROM emp e, dept d
  2  WHERE e.deptno = d.deptno
  3  AND e.ename='SCOTT';
SCOTT@linora> select sql_text,sql_id,hash_value,child_number from v$sql
  2  where sql_text like 'SELECT /*+ display_cursor_example */%';
SQL_TEXT                                                     SQL_ID        HASH_VALUE CHILD_NUMBER
------------------------------------------------------------ ------------- ---------- ------------
SELECT /*+ display_cursor_example */ * FROM emp e, dept d WH 7p8g08wnrjn43  695783555            0
ERE e.deptno = d.deptno AND e.ename='SCOTT'
```
查看此语句的执行计划：
```
SCOTT@linora> set pagesize 9999
SCOTT@linora> col PLAN_TABLE_OUTPUT for a100
SCOTT@linora> select * from table(dbms_xplan.display_cursor('7p8g08wnrjn43','0','advanced'));
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------
SQL_ID  7p8g08wnrjn43, child number 0
-------------------------------------
SELECT /*+ display_cursor_example */ * FROM emp e, dept d WHERE
e.deptno = d.deptno AND e.ename='SCOTT'

Plan hash value: 3625962092
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |       |       |     4 (100)|          |
|   1 |  NESTED LOOPS                |         |     1 |    58 |     4   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |         |     1 |    58 |     4   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | EMP     |     1 |    38 |     3   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PK_DEPT |     1 |       |     0   (0)|          |
|   5 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     1 |    20 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
   1 - SEL$1
   3 - SEL$1 / E@SEL$1
   4 - SEL$1 / D@SEL$1
   5 - SEL$1 / D@SEL$1
Outline Data
------------
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.4')
      DB_VERSION('11.2.0.4')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      FULL(@"SEL$1" "E"@"SEL$1")
      INDEX(@"SEL$1" "D"@"SEL$1" ("DEPT"."DEPTNO"))
      LEADING(@"SEL$1" "E"@"SEL$1" "D"@"SEL$1")
      USE_NL(@"SEL$1" "D"@"SEL$1")
      NLJ_BATCHING(@"SEL$1" "D"@"SEL$1")
      END_OUTLINE_DATA
  */
Predicate Information (identified by operation id):
---------------------------------------------------
   3 - filter("E"."ENAME"='SCOTT')
   4 - access("E"."DEPTNO"="D"."DEPTNO")
Column Projection Information (identified by operation id):
-----------------------------------------------------------
   1 - "E"."EMPNO"[NUMBER,22], "E"."ENAME"[VARCHAR2,10], "E"."JOB"[VARCHAR2,9],
       "E"."MGR"[NUMBER,22], "E"."HIREDATE"[DATE,7], "E"."SAL"[NUMBER,22],
       "E"."COMM"[NUMBER,22], "E"."DEPTNO"[NUMBER,22], "D"."DEPTNO"[NUMBER,22],
       "D"."DNAME"[VARCHAR2,14], "D"."LOC"[VARCHAR2,13]
   2 - "E"."EMPNO"[NUMBER,22], "E"."ENAME"[VARCHAR2,10], "E"."JOB"[VARCHAR2,9],
       "E"."MGR"[NUMBER,22], "E"."HIREDATE"[DATE,7], "E"."SAL"[NUMBER,22],
       "E"."COMM"[NUMBER,22], "E"."DEPTNO"[NUMBER,22], "D".ROWID[ROWID,10],
       "D"."DEPTNO"[NUMBER,22]
   3 - "E"."EMPNO"[NUMBER,22], "E"."ENAME"[VARCHAR2,10], "E"."JOB"[VARCHAR2,9],
       "E"."MGR"[NUMBER,22], "E"."HIREDATE"[DATE,7], "E"."SAL"[NUMBER,22],
       "E"."COMM"[NUMBER,22], "E"."DEPTNO"[NUMBER,22]
   4 - "D".ROWID[ROWID,10], "D"."DEPTNO"[NUMBER,22]
   5 - "D"."DNAME"[VARCHAR2,14], "D"."LOC"[VARCHAR2,13]

67 rows selected.
```
如果display_cursor不添加前面两个参数，则表示查看刚刚执行过的SQL的执行计划。如：
```
SCOTT@linora> SELECT * FROM emp e, dept d WHERE e.deptno = d.deptno;
SCOTT@linora> select * from table(dbms_xplan.display_cursor('','','advanced'));
```
#### 2.3 DISPLAY_AWR
语法：
```
DBMS_XPLAN.DISPLAY_AWR( 
   sql_id            IN      VARCHAR2,
   plan_hash_value   IN      NUMBER DEFAULT NULL,
   db_id             IN      NUMBER DEFAULT NULL,
   format            IN      VARCHAR2 DEFAULT TYPICAL);
```
如果某一条语句的执行计划已经从shared pool清除了，那么此时想要查看此SQL的执行计划，就只能从display_awr中查看了，通过display_awr获取的SQL执行计划来自dba_hist_sql_plan，但display_awr不能查看执行步骤中对应的谓词条件！
```
SCOTT@linora> SELECT * FROM emp e, dept d WHERE e.deptno = d.deptno;
SCOTT@linora> select sql_text,sql_id,hash_value,child_number from v$sql
  2  where sql_text like 'SELECT * FROM emp%';
SQL_TEXT                                                     SQL_ID        HASH_VALUE CHILD_NUMBER
------------------------------------------------------------ ------------- ---------- ------------
SELECT * FROM emp e, dept d WHERE e.deptno = d.deptno        8mfyh7m4tph6q 3461957154            0

SCOTT@linora> exec dbms_workload_repository.create_snapshot();
PL/SQL procedure successfully completed.

SCOTT@linora> alter system flush shared_pool;
System altered.

SCOTT@linora> select * from table(dbms_xplan.display_cursor('8mfyh7m4tph6q','0','advanced'));
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------
SQL_ID: 8mfyh7m4tph6q, child number: 0 cannot be found

SCOTT@linora> select * from table(dbms_xplan.display_awr('8mfyh7m4tph6q'));
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------
SQL_ID 8mfyh7m4tph6q
--------------------
SELECT * FROM emp e, dept d WHERE e.deptno = d.deptno
Plan hash value: 844388907
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |       |       |     6 (100)|          |
|   1 |  MERGE JOIN                  |         |    14 |   812 |     6  (17)| 00:00:01 |
|   2 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     4 |    80 |     2   (0)| 00:00:01 |
|   3 |    INDEX FULL SCAN           | PK_DEPT |     4 |       |     1   (0)| 00:00:01 |
|   4 |   SORT JOIN                  |         |    14 |   532 |     4  (25)| 00:00:01 |
|   5 |    TABLE ACCESS FULL         | EMP     |    14 |   532 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
```
需要注意的是，如果某目标SQL的执行计划已经不在shared pool中了，SQL的执行计划已经被Oracle捕获并且存储到了AWR的Repository中，才可以使用display_awr，且版本是10g以上；如果是9i，则需要部署statspack，且采集的level必须大于6才可查看历史SQL的执行计划。
### 3. auto trace
语法：
```
SCOTT@linora> set autotrace -h
Usage: SET AUTOT[RACE] {OFF | ON | TRACE[ONLY]} [EXP[LAIN]] [STAT[ISTICS]]
```
<li><code>set autot on/off</code>在当前sessions完全打开/关闭autotrace，同时输出结果及执行计划和资源消耗</li>
```
SCOTT@linora> set autot on
SCOTT@linora> SELECT e.ename,e.job,e.sal,d.dname FROM emp e, dept d 
WHERE e.deptno = d.deptno and e.ename='SCOTT';

ENAME      JOB              SAL DNAME
---------- --------- ---------- --------------
SCOTT      ANALYST         3000 RESEARCH

Execution Plan
----------------------------------------------------------
Plan hash value: 3625962092
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |     1 |    34 |     4   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |         |     1 |    34 |     4   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |         |     1 |    34 |     4   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | EMP     |     1 |    21 |     3   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PK_DEPT |     1 |       |     0   (0)| 00:00:01 |
|   5 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     1 |    13 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------

   3 - filter("E"."ENAME"='SCOTT')
   4 - access("E"."DEPTNO"="D"."DEPTNO")
Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
          9  consistent gets
          0  physical reads
          0  redo size
        743  bytes sent via SQL*Net to client
        519  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
```
<li><code>set autot trace</code>只输出目标SQL执行计划和资源消耗，对于SQL执行结果则只显示执行结果的数量</li>
```
SCOTT@linora> set autot trace
SCOTT@linora> SELECT e.ename,e.job,e.sal,d.dname FROM emp e, dept d 
WHERE e.deptno = d.deptno and e.ename='SCOTT';

Execution Plan
----------------------------------------------------------
Plan hash value: 3625962092
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |     1 |    34 |     4   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |         |     1 |    34 |     4   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |         |     1 |    34 |     4   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | EMP     |     1 |    21 |     3   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PK_DEPT |     1 |       |     0   (0)| 00:00:01 |
|   5 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     1 |    13 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   3 - filter("E"."ENAME"='SCOTT')
   4 - access("E"."DEPTNO"="D"."DEPTNO")
Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
          9  consistent gets
          0  physical reads
          0  redo size
        743  bytes sent via SQL*Net to client
        519  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
```
<li><code>set autot trace exp</code>只输出SQL执行计划，而不会显示目标SQL的执行结果和资源消耗</li>
```
SCOTT@linora> set autot trace exp
SCOTT@linora> SELECT e.ename,e.job,e.sal,d.dname FROM emp e, dept d 
WHERE e.deptno = d.deptno and e.ename='SCOTT';

Execution Plan
----------------------------------------------------------
Plan hash value: 3625962092
----------------------------------------------------------------------------------------
| Id  | Operation                    | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |         |     1 |    34 |     4   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                |         |     1 |    34 |     4   (0)| 00:00:01 |
|   2 |   NESTED LOOPS               |         |     1 |    34 |     4   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | EMP     |     1 |    21 |     3   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PK_DEPT |     1 |       |     0   (0)| 00:00:01 |
|   5 |   TABLE ACCESS BY INDEX ROWID| DEPT    |     1 |    13 |     1   (0)| 00:00:01 |
----------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   3 - filter("E"."ENAME"='SCOTT')
   4 - access("E"."DEPTNO"="D"."DEPTNO")
```
<li><code>set autot trace stat</code>只输出SQL执行结果资源消耗，而不显示执行计划</li>
```
SCOTT@linora> set autot trace stat
SCOTT@linora> SELECT e.ename,e.job,e.sal,d.dname FROM emp e, dept d 
WHERE e.deptno = d.deptno and e.ename='SCOTT';

Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
          9  consistent gets
          0  physical reads
          0  redo size
        743  bytes sent via SQL*Net to client
        519  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
```
### 4. 10046事件
请参照前文[Oracle追踪SQL的方法](/oracle-trace.html) 10046及tkprof的相关介绍。
### 5. 总结
在以上四种方法中，都可以看到SQL的执行计划，但除了10046外，其他方法获得的执行计划，都有可能是不准确的。因此，如果要获得SQL的真实执行计划，最好使用10046事件进行跟踪(SQL_TRACE也是10046的一个级别)。
#### explain plan 
对于此方法而言，目标SQL根本就没有被执行过，因此，该执行计划极有可能是不准确的，特别是含有绑定变量的情况下，针对于bind peeking，Oracle可能会根据绑定变量窥视进行执行计划的调整。
#### dbms_xplan
除了dbms_xplan.display执行计划可能不准确外，dbms_xplan.display_awr，dbms_xplan.display_cursor都是准确的执行计划，因为后面两个都表示目标SQL被真正执行过。
#### set autotrace
autotrace设置为on或者traceonly时，目标SQL已经被实际执行过了，但当使用set autot trace exp时，如果执行的是select语句，则该SQL不会被Oracle执行，如果是DML修改，此时的SQL是会被实际执行的。虽然使用auto trace on/traceonly目标SQL都会被执行，但是用这种方法得到的执行计划还有可能是不准确的，因为使用auto trace命令所显示的执行计划都是源于explain plan的调用，跟TOAD和PL/SQL Developer一样。  
 <code><B>要获得真实的执行计划，尽量采用10046事件或者dbms_xplan.display_cursor！！！</B></code>  
 
 Reference:  
[基于Oracle的SQL优化](http://www.dbsnake.net/books)  
