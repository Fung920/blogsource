---
layout: post
title: "12C CDB简介"
date: 2014-08-08 11:55:15
comments: false
categories: oracle
tags: 12c
keywords: 12c new feature,PDB,CDB,Multitenant Environment
description: 12c new feature Multitenant Environment,12c新特性--多租户环境(Pluggable database)
---
2012年9月，Larry Ellison与旧金山OOW中透露12c的最重要的新特性：Pluggable Database。随后于2013年六月发布Linux 12c版本。
<!--more-->
### 1. 关于多租户环境
Oracle database 12c包含了过百项的新特色，其中最引人注目的还是Pluggable Database，中文翻译为可插拔数据库或者叫多租户环境。在过去的数据库服务器上，比较常见的是一台服务器只包含一个数据库或实例(RAC)，然后很多情况下，这种结构对资源的利用不够经济。在一些硬体条件足够的环境下，一台服务器可能支持多个数据库，或者一个数据库以schema分开对多个应用系统提供服务。但以上方案均有缺点：
<li>主机数据库一对一</li>
软硬件资源无法共享，造成资源浪费；  
每个DB都要存储一份Oracle DB提供的组件，如DBMS_XXX，因此，额外带来存储的开销。
<li>一台主机对应多个数据库</li>
每个数据库需要对应的实例，因此，内存无法由Oracle管理系统自动调用；  
每个DB都要存储一份Oracle DB提供的组件，如DBMS_XXX，因此，额外带来存储的开销。
<li>以schema方式提供多个应用服务</li>
各个应用的资料无法独立管理，且schema命名须规范，避免重复；  
无法将一个应用资料快速转移到其他数据库。  
为了解决资源共享而管理独立的问题，Oracle从12C开始推出多租户环境，即可插拔数据库。
### 2. 多租户架构
Oracle多租户环境主要有两个组成部分：多租户容器数据库(multitenant container database, CDB)及可插拔数据库(pluggable database, PDB)。
![multitenant](/images/multitenant.png)
CDB是由一个数据库组成，可由一个或者多个instance(RAC)管理，如上图所示，每个CDB可包含root container(CDB$ROOT)，seed pluggable database container (PDB$SEED)和pluggable database container(PDBs)。
<li>Root</li>
每个CDB有且只能有一个root container。CDB$ROOT存储Oracle提供的元数据(如数据字典或者Oracle提供的pl/sql包)及common user的账户资料，这个common user是每个容器都存在的用户，它可登入管理每个它有适当权限的PDB。
<li>Seed</li>
Seed是创建PDB的模版，在每个CDB中，有且仅有一个，在PDB$SEED中，用户不能添加及修改其中对象。
<li>PDBs</li>
每个CDB至多个包含252个PDB。PDB对于普通用户及开发人员来说，和以前版本的数据库没什么区别，从这个角度来说，如果某个PDB要搬迁至其他主机，并不需要进行任何修改，只需要将PDB搬迁至其他CDB即可。被拔除的PDB以一个XML文件形式存在，记录了此PDB相关的信息。  
<li>总结</li>
1.PDB允许我们快速建立空数据库(create pdb from pdb$seed)  
2.将non-CDB数据库导入PDB(使用Export/Import或者DBMS_PDB)  
3.快速复制PDB(Clone PDB)  
4.快速搬迁PDB至其他CDB(unplug & plug-in pdb)  
每个PDB是单独的数据库，相互独立存在(但在同一个CDB中，共用同一个实例)，Oracle将本身的数据字典和PDB上的数据字典彻底分开，分别存放于root，seed及PDB中，因此，每一个PDB将系统本身的资料，数据字典及程序代码等单独存放在各自的PDB中，这样就能方便管理者快速拔除及插入。同时，每一个PDB都会有自己的system表空间，用于存放本身相关的数据字典等元数据。而由Oracle本身提供的，各个PDB共享的元数据及物件，则存入CBD$ROOT的数据字典中。在升级数据库的时候，只需要升级CDB$ROOT，其下所有的PDB会一起更新。  

<b>To be continued!</b>

Reference：  
[Oracle® Database New Features Guide 12c Release 1 (12.1)](http://docs.oracle.com/database/121/NEWFT/toc.htm)  
[Managing a Multitenant Environment](http://docs.oracle.com/database/121/ADMIN/part_cdb.htm#ADMIN13506)
