---
title: 使用SPM固定执行计划
categories: oracle
comments: false
date: 2019-08-08 00:24:48
tags: [spm, tuning]
---
数据库在运行过程中，会由于各种原因的变化，存在执行计划不稳定的情况。Oracle 11g开始有SQL Plan Management来管理，稳定执行计划。
一般执行计划不稳定有以下几种原因引起（包括但不限于）：
* 数据倾斜
* 统计信息不准确
* 数据库升级

<!--more-->
## 1. 绑定变量带来的问题
从Oracle 9i开始，引进了绑定变量窥视（bind peeking）新特性，在对数据倾斜的绑定变量中，在ORACLE第一次解析SQL时会将变量的真实值代入产生执行计划，以后对所有的同样的绑定变量SQL都采用这个执行计划了。这种特性在大部分情况下能结合绑定变量减少SQL的解析时间，但对存在数据倾斜的SQL有可能产生极其糟糕的执行计划。
在11g开始，引进自适应游标共享（Adaptive Cursor Sharing，ACS），对一个绑定变量生成多个子执行计划，以求减低数据倾斜对执行计划的影响,从而达到动态调整执行计划的目的。

### 1.1 绑定变量相关视图

`v$sql`中新增IS_BIND_SENSITIVE和IS_BIND_AWARE字段：
```sql
select is_bind_sensitive, is_bind_aware, sql_id, child_number
from v$sql
where sql_id = '&1';
```
如果游标中存在绑定变量，数据库会根据传入的实际值判断不同的值能否会影响执行计划，如果会，则游标被标记为`Bind-Sensitive`。在v$sql视图中的IS_BIND_SENSITIVE值则为Y。
当该SQL被执行过几次后，数据库会根据传入的实际值来决定是否需要修改执行计划，如果需要，则该游标是`Bind-Aware`，在v$sql视图中`IS_BIND_AWARE`也被标记为Y。

`V$SQL_CS_SELECTIVITY`视图则展示了不同值的selectivity。
`V$SQL_CS_STATISTICS`视图展示了标记为`Bind-Sensitive`和`Bind-Aware`游标的一些统计信息，如内存读，CPU时间等。
```sql
select  child_number,
bind_set_hash_value,
peeked,
executions,
rows_processed,
buffer_gets,
cpu_time
from v$sql_cs_statistics
where sql_id = '&1';
```

### 1.2 查询绑定变量值

* 从AWR查找

```sql
col name for a30
col value_string for a30
set line 200 pagesize 9999
select SNAP_ID,name,datatype_string,value_string,datatype from DBA_HIST_SQLBIND where sql_id='&1'

set line 200 pagesize 9999
col bind1 for a30
col bind2 for a30
col bind3 for a30
col bind4 for a30
select
snap_id,SQL_ID,PLAN_HASH_VALUE,
dbms_sqltune.extract_bind(bind_data,1).value_string bind1,
dbms_sqltune.extract_bind(bind_data,2).value_string bind2,
dbms_sqltune.extract_bind(bind_data,3).value_string bind3,
dbms_sqltune.extract_bind(bind_data,4).value_string bind4
from dba_hist_sqlstat
where sql_id = '&1'
order by snap_id;
```

* 从cursor中查找

```sql
select ADDRESS
      ,HASH_VALUE
      ,CHILD_NUMBER
      ,name
      ,DATATYPE_STRING
      ,VALUE_STRING
      ,LAST_CAPTURED
from v$sql_bind_capture where sql_id ='&1';
```

### 1.3 从共享内存中删除游标

* 通过SQL ID查找SQL地址及hash值

```sql
--查找sql的address和hash_value
select address, hash_value, sql_text
from v$sqlarea
where sql_id='&1';
```

* 调用dbms_shared_pool清除游标

将产生的address及hash_value代入 &1， &2, 其中，c代表需要purge的类型是cursor，r表示trigger,q表示sequence等.

```sql
exec dbms_shared_pool.purge('&1,&2','C');
```

## 2. 如何使用SPM管理执行计划
### 2.1 相关术语
* SPM
    SPM是管理SQL执行计划的框架。其主要目的是防止由于执行计划改变而导致的SQL性能的退化。同时也提供了动态调整SQL执行计划的框架。

* SQL Plan Baseline
    当一个新的执行计划产生时候，SPM不一定会立即启用它，只有确认这些执行计划不会带来性能下降或者能提升性能，SPM才会采用(Accepted),而这些accepted的执行计划则被成为baseline(以下简称基线)。

* Plan Evolution
    把accepted的执行计划加入到基线的过程。

* SQL Management Base(SMB)
    存储基线、执行计划历史等的数据字典。

### 2.2 工作原理
SPM通过两个动态初始化参数进行控制, 两个参数均为PDB级别可修改。
* optimizer_capture_sql_plan_baselines
    自动识别重复的SQL语句，并且为这些语句生成基线(__从另一个侧面说明，SPM需要结合绑定变量使用，如果不使用绑定变量，建议使用SQL profile+ force_match参数__)。该参数默认为false。
* optimizer_use_sql_plan_baselines
    启用或者关闭SMB中的基线，默认为true。当启用时，优化器会从SMB中查找相应的基线并且挑选cost最小的作为该SQL的执行计划;如果是新执行计划，数据库会自动把这些新计划以unaccepted状态加入到基线中。

其工作流程如下图所示：
<img width=500 src="/images/spm_flowchart.jpg" >

如果存在基线，优化器根据新产生的执行计划是否在基线中而作出不同选择：
* 如果新计划在基线中，则会执行此计划
* 如果新计划不在基线中，优化器会把新产生的执行计划标记为accepted，且加入到plan history中，接下来优化器会根据以下情况作出不同选择：
   - 如果fixed plan存在于基线中，优化器会使用最低代价的fixed plan
   - 如果没有fixed plan，优化器挑选基线中代价最低的
   - 如果基线中不存在reproduced的执行计划，比如所有执行计划相关的索引都drop掉了，优化器会采用新产生的执行计划

### 2.3 创建基线
创建基线有以下几种方式：

* 使用SQL Tuning Set(STS)
* 从缓存/AWR中加载(new for 12.2)
* 从其他库导出并导入
* 自动收集(optimizer_capture_sql_plan_baselines=TRUE)

### 2.3.1 从缓存中进行加载

* 游标中已经存在较优的执行计划:

```sql
declare
  pls number;
begin
  pls := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(sql_id  => '&1',
                                               --plan_hash_value => &2,
                                               enabled  => 'YES');
end;
/

set serveroutput on

var n number
begin
:n:=dbms_spm.load_plans_from_cursor_cache(sql_id=>'&sql_id', plan_hash_value=>&plan_hash_value, fixed =>'NO', enabled=>'YES');
end;
/

EXEC dbms_output.put_line('Number of plans loaded: ' || :n);
```

* 通过filter筛选SQL

```
exec :pls := dbms_spm.load_plans_from_cursor_cache( -
           attribute_name => 'SQL_TEXT', -
           attribute_value => '&1');
```

### 2.3.2 从AWR中加载
```sql
exec pls := dbms_spm.load_plans_from_awr(begin_snap => &1, end_snap => &2);
```

### 2.4 查询基线
通过以下视图查询基线状态:
```sql
select sql_text, plan_name, enabled, accepted from dba_sql_plan_baselines;

--通过sql_id查找基线
col sql_handle for a30
col plan_name for a50
set line 200 pagesize 9999
select s.sql_id, s.plan_hash_value, b.sql_handle, b.plan_name,
b.signature, b.enabled, b.accepted, b.fixed, s.sql_text
from v$sqlarea s JOIN dba_sql_plan_baselines b
on (s.exact_matching_signature = b.signature) and s.sql_id = '&1';

--查找仅来源于手动加载的基线
col sql_handle for a30
col plan_name for a30
col creator for a30
set line 200 pagesize 200
select sql_handle, plan_name, origin, enabled, accepted,
fixed,creator,optimizer_cost,sql_text
from dba_sql_plan_baselines where origin = 'MANUAL-LOAD';
```

`dba_sql_plan_baselines`中栏位的含义:
如果是Enabled=No或者是Reproduced=No, 则优化器不会考虑相关的执行计划。
![SQL Baseline](/images/baseline1.jpg)

可通过__`dbms_spm.alter_sql_plan_baseline`__去调整这些参数值。如:
```sql
var temp varchar2(4000)
exec :temp := dbms_spm.alter_sql_plan_baseline(sql_handle=>'&SQL_HANDLE',plan_name=>'&SQL_PLAN',attribute_name=>'enabled',attribute_value=>'YES');

var pbsts varchar2(30);
exec :pbsts := dbms_spm.alter_sql_plan_baseline('&SQL_HANDLE','&SQL_PLAN','accepted','NO');

SET SERVEROUTPUT ON
DECLARE
  l_plans_altered  PLS_INTEGER;
BEGIN
  l_plans_altered := DBMS_SPM.alter_sql_plan_baseline(
    sql_handle      => '&SQL_HANDLE',
    plan_name       => '&SQL_PLAN',
    attribute_name  => 'fixed',
    attribute_value => 'YES');

  DBMS_OUTPUT.put_line('Plans Altered: ' || l_plans_altered);
END;
/
```


查询基线执行计划:
```sql
SELECT PLAN_TABLE_OUTPUT
FROM   V$SQL s, DBA_SQL_PLAN_BASELINES b,
       TABLE(
       DBMS_XPLAN.DISPLAY_SQL_PLAN_BASELINE(b.sql_handle,b.plan_name,'all')
       ) t
WHERE  s.EXACT_MATCHING_SIGNATURE=b.SIGNATURE
AND    b.PLAN_NAME=s.SQL_PLAN_BASELINE
AND    s.SQL_ID='&SQL_ID';
```

### 2.5 发展基线
Plan Evolution(发展基线)就是优化器识别新执行计划(unaccepted)并加入到基线的过程。
创建发展基线的大致步骤（12c）：
* Create Evolve Task (dbms_spm.create_evolve_task)
* Execute Evolve Task (dbms_spm.execute_evolve_task)
* Report Evolve Task (dbms_spm.report_evolve_task)
* Accept Recommendation (dbms_spm.accept_sql_plan_baseline)

#### 2.5.1 手工创建发展基线(12c)
* 创建evolve任务

```sql
VARIABLE cnt NUMBER
VARIABLE tk_name VARCHAR2(50)
VARIABLE exe_name VARCHAR2(50)
VARIABLE evol_out CLOB

EXECUTE :tk_name := DBMS_SPM.CREATE_EVOLVE_TASK(
  sql_handle => '&SQL_HANDLE',
  plan_name  => '&SQL_PLAN');

SELECT :tk_name FROM DUAL;
```

* 执行该任务

```sql
EXECUTE :exe_name :=DBMS_SPM.EXECUTE_EVOLVE_TASK(task_name=>:tk_name);
SELECT :exe_name FROM DUAL;
```

* 查看报告

```sql
EXECUTE :evol_out := DBMS_SPM.REPORT_EVOLVE_TASK( task_name=>:tk_name, execution_name=>:exe_name );
SELECT :evol_out FROM DUAL;
```

* 实施推荐

```sql
EXECUTE :cnt := DBMS_SPM.IMPLEMENT_EVOLVE_TASK( task_name=>:tk_name, execution_name=>:exe_name );
```

#### 2.5.2 手动发展基线(11g)
```sql
var report clob;
exec :report := dbms_spm.evolve_sql_plan_baseline('&SQL_HANDLE');
print :report
```

#### 2.5.3 自动发展基线(12c)
通过`SYS_AUTO_SPM_EVOLVE_TASK`可以在12c中自动发展基线，

查看当前自动任务配置信息：
```sql
COLUMN parameter_name FORMAT A25
COLUMN parameter_value FORMAT a25

SELECT parameter_name, parameter_value
FROM   dba_advisor_parameters
WHERE  task_name = 'SYS_AUTO_SPM_EVOLVE_TASK'
AND    parameter_value != 'UNUSED'
ORDER BY parameter_name;
```

关闭自动任务:
```sql
BEGIN
  DBMS_SPM.set_evolve_task_parameter(
    task_name => 'SYS_AUTO_SPM_EVOLVE_TASK',
    parameter => 'ACCEPT_PLANS',
    value     => 'FALSE');
END;
/
```

查看自动任务执行结果：
```sql
SET LONG 1000000 PAGESIZE 1000 LONGCHUNKSIZE 100 LINESIZE 100
SELECT DBMS_SPM.report_auto_evolve_task FROM  dual;
```



### 2.6 删除基线
```sql
--指定某一个baseline删除
dbms_spm.drop_sql_plan_baseline(sql_handle=>'&SQL_HANDLE',plan_name=>'&SQL_PLAN')

--根据sql handle删除
DECLARE
  v_dropped_plans number;
BEGIN
  v_dropped_plans := DBMS_SPM.DROP_SQL_PLAN_BASELINE (
     sql_handle => '&SQL_HANDLE'
);
  DBMS_OUTPUT.PUT_LINE('dropped ' || v_dropped_plans || ' plans');
END;
/

--或者指定plan name删除
SET SERVEROUTPUT ON
DECLARE
  l_plans_dropped  PLS_INTEGER;
BEGIN
  l_plans_dropped := DBMS_SPM.drop_sql_plan_baseline (
    sql_handle => NULL,
    plan_name  => '&SQL_PLAN');
  DBMS_OUTPUT.put_line(l_plans_dropped);
END;
/
```

### 2.7 管理SBM
* 查询当前SBM配置

```sql
SELECT PARAMETER_NAME, PARAMETER_VALUE FROM DBA_SQL_MANAGEMENT_CONFIG;

PARAMETER_NAME					   PARAMETER_VALUE
-------------------------------------------------- ---------------
SPACE_BUDGET_PERCENT						10
PLAN_RETENTION_WEEKS						53
```
上述结果为默认配置，SBM可使用的空间上限为sysaux的10%，保留期限为53周。



* 修改SBM配置

```sql
--修改空间限制为30%:
EXECUTE DBMS_SPM.CONFIGURE('space_budget_percent',30);

--修改SBM保留期限, 修改为105周，默认为53周
EXECUTE DBMS_SPM.CONFIGURE('plan_retention_weeks',105);
```



</BR>
Reference:
[SQL Plan Management with Oracle Database 12c Release 2](https://www.oracle.com/technetwork/database/bi-datawarehousing/twp-sql-plan-mgmt-12c-1963237.pdf)
[Managing SQL Plan Baselines](https://docs.oracle.com/database/121/TGSQL/tgsql_spm.htm#TGSQL94621)
[How to Load SQL Plans into SQL Plan Management (SPM) from the Automatic Workload Repository (AWR) (Doc ID 789888.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=381083850018605&id=789888.1&displayIndex=10&_afrWindowMode=0&_adf.ctrl-state=sc2p27a0x_77#GOAL)
[White Papers and Blog Entries for Oracle Optimizer (Doc ID 1337116.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=381244985698664&id=1337116.1&displayIndex=14&_afrWindowMode=0&_adf.ctrl-state=sc2p27a0x_216#aref_section31)
[How to Use SQL Plan Management (SPM) - Plan Stability Worked Example (Doc ID 456518.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=381333370628274&parent=DOCUMENT&sourceId=1905305.1&id=456518.1&_afrWindowMode=0&_adf.ctrl-state=sc2p27a0x_322)

__EOF__


