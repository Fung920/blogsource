---
layout: post
title: "Oracle表连接类型"
date: 2014-08-17 21:12:10
comments: false
categories: oracle
tags: tuning
keywords: Oracle 表连接类型
description: 表的内连接，外连接
---
在Oracle优化器中，多表的连接顺序，连接方法的不同，对CBO产生的COST有很大的差异。在优化器解析含表连接的SQL时，它会根据目标SQL的写法来决定表连接的顺序、表连接的方法和单表访问路径来决定此SQL的最终执行计划。
<!--more-->
Oracle表之间的连接总的分为内连接(Inner Join)和外连接(Outer Join)，外连接又包含左外连接(Left Outer Join)，右外连接(Right Outer Join)和全外连接(Full Outer Join)。  
![SQL JOIN](/images/sql_joins.png)
### 1. 内连接
在标准SQL和Oracle中，默认的连接都为Inner Join，因此，在写SQL的时候，Inner Join是不需要明确指出来的。内连接是指多表连接的连接结果只包含那些满足连接条件的记录。  
```
#创建测试表
FUNG@linora> create table t1 (id number,name varchar2(20));
Table created.
FUNG@linora> create table t2 (id number,job varchar2(10));
Table created.
FUNG@linora> insert into t1 values(1,'fung');
insert into t1 values(2,'kong');
insert into t1 values(3,'kyun');
insert into t1 values(4,'jordan');
insert into t2 values(1,'dba');
insert into t2 values(2,'sa');
insert into t2 values(3,'hr');
insert into t2 values(4,'dev');
insert into t2 values(5,'mgr');
insert into t1 values(6,'pippen');
commit;
FUNG@linora> select * from t1;
        ID NAME
---------- --------------------
         1 fung
         2 kong
         3 kyun
         4 jordan
         6 pippen
FUNG@linora> select * from t2;
        ID JOB
---------- ----------
         1 dba
         2 sa
         3 hr
         4 dev
         5 mgr
FUNG@linora> select t1.id,t1.name,t2.job from t1,t2 where t1.id=t2.id;
        ID NAME                 JOB
---------- -------------------- ----------
         1 fung                 dba
         2 kong                 sa
         3 kyun                 hr
         4 jordan               dev
```
上述SQL改写为标准SQL为
```
select t1.id,t1.name,t2.job from t1 join t2 on (t1.id=t2.id);
--或者
select id,t1.name,t2.job from t1
join t2 using (id);
```
还有一种特殊的JOIN Using叫自然连接(Natural Join)，自然连接的含义是表连接的连接列是表连接的两个表所有的<b>同名列</b>。如
```
select id,t1.name,t2.job
from t1
natural join t2;
```
在现实中，并不推荐自然连接的写法，因为列名相同，但其含义不一定相同，使用自然连接会导致目标SQL结果出错的风险(包括同名列数据类型不一样也会报错)。
### 2. 外连接
外连接指表连接的连接结果除了包含完全满足连接条件的外，还包含驱动表中所有不满足连接条件的记录。外连接的SQL写法中，可以忽略outer关键字。
#### 2.1 左外连接
```
FUNG@linora> select t1.id,t1.name,t2.job from t1
left outer join t2 on (t1.id=t2.id);
        ID NAME                 JOB
---------- -------------------- ----------
         1 fung                 dba
         2 kong                 sa
         3 kyun                 hr
         4 jordan               dev
         6 pippen
```
等价于(Oracle SQL写法)：
```
select t1.id,t1.name,t2.job from t1,t2 where t1.id=t2.id(+);
```
在上例中，驱动表是位于关键字'left outer join'左边的T1表，因此，返回的结果不仅包含符合连接条件的所有记录，还包含了T1表所有不符合连接条件的记录。Oracle的'+'号写法，是要把'+'号放在被驱动表栏位。对于被驱动表中不符合连接条件的查询记录，以NULL补充(关键字'+'出现在哪个表的连接列后面，就表明哪个表会以NULL来填充那些不满足连接条件并位于该表的查询列)。如果需要查找驱动表符合连接条件外的其他记录，可以在外连接上添加一个where子句，'where t2.id is NULL'。
#### 2.2 右外连接
```
FUNG@linora> select t1.id,t1.name,t2.job from t1
right outer join t2 on (t1.id=t2.id);

        ID NAME                 JOB
---------- -------------------- ----------
         1 fung                 dba
         2 kong                 sa
         3 kyun                 hr
         4 jordan               dev
                                mgr
```
等价于(Oracle SQL写法)：
```
select t1.id,t1.name,t2.job from t1,t2 where t1.id(+)=t2.id
```
对于左外和右外的解释，可以看文章开始的图片，对于两张表内连接，在数学上可以视为两者的交集，左/右外连接，是相对于交集的左/右边而言。
#### 2.3 全外连接
```
FUNG@linora> select t1.id,t1.name,t2.job from t1
full outer join t2 on (t1.id=t2.id);
        ID NAME                 JOB
---------- -------------------- ----------
         1 fung                 dba
         2 kong                 sa
         3 kyun                 hr
         4 jordan               dev
                                mgr
         6 pippen
```
可以看到，全外连接查询的结果包含了满足连接条件的记录和不满足连接条件的记录(包括T1和T2表)，不满足条件的记录所对应的另外一个表中的查询列会以NULL填充。



Reference:  
[基于Oracle的SQL优化](http://www.dbsnake.net/books)  