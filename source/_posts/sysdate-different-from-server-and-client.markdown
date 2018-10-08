---
layout: post
title: "Sysdate Different From Server and Client"
date: 2015-06-11 18:29:18
comments: false
categories: oracle
tags: 
keywords: dbtimezone
description: sysdate different from server and client
---
某客户最近搬迁，在支援的过程中发现几点很好玩的东西。现在记录下来。
<!--more-->
### 1.AIX RAC共享磁盘属性设置错误导致OCR只能在一个节点启动
这套系统通过存储镜像进行LUN迁移，主机不变，仅仅是更换机房。然而，在启动集群的时候，报
```
CRS-1714:Unable to discover any voting files, retrying discovery in 15 seconds
```
第一反应就是ocr文件损坏了，但是仔细一看，一个节点已经运行正常了，报错的是另一个节点，就是说不管哪个节点起集群，都能run，但是另一节点就加入不了集群。怀疑共享磁盘属性设置有问题，一查看：
```
lsattr -El hdisk12|grep -i reserve
reserve_policy  single_path     Reserve Policy    True
```
果真是，建议客户备份数据库，同时，不建议通过<code>chdev</code>直接修改磁盘属性，可以考虑通过删除磁盘组磁盘，再修改属性，再加进去。但是后来客户自己直接修改了磁盘属性，集群目前运行正常。
### 2.11g grid infrastructure时区配置文件导致sysdate输出不一致
这套系统也通过存储镜像进行迁移，但是更换了主机，同时存在10g和11g数据库。用<code>select sysdate from dual</code>查看时间，11g的服务端和客户端时间返回不一致，通过<code>select dbtimezone from dual</code>发现时区为+8:00，OS为AIX 7.1，时区为BEIST-8.
AIX7已经没有BEIST-8的时区了，建议客户修改为Asia/Shanghai，修改完后还是不正确。这套11g的系统是用ASM管理的，就意味着存在GI，查看MOS [ID 1209444.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1209444.1)，里面有提及
```
2. For 11.2.0.2 and above, TZ entry in $GRID_HOME/crs/install/s_crsconfig_<nodename>_env.txt sets to correct time zone.
```
查看该文件，发现TZ=CST6CDT，估计安装GI的时候OS时区不正确导致，修改成Asia/Shanghai，重启CRS，sysdate返回结果正常。
```
[root@oel6 ~]# cat `find / -name s_crsconfig_$HOSTNAME*.txt -print`
```
### 3.模拟sysdate server client返回不一致：
```
[grid@oel6:/home/grid]$ grep TZ `find $ORACLE_HOME -name s_crsconfig_$HOSTNAME*.txt`
TZ=CST6CDT
```
服务器上查询：
```
[grid@oel6:/home/grid]$ cat /etc/sysconfig/clock 
ZONE="Asia/Shanghai"
[oracle@oel6:/home/oracle]$ date
Thu Jun 11 19:03:20 CST 2015
[oracle@oel6:/home/oracle]$ exit
SQL> alter session set nls_date_format='yyyy-dd-mm hh24:mi:ss';
Session altered.
SQL> select sysdate,dbtimezone,current_date from dual;
SYSDATE             DBTIME CURRENT_DATE
------------------- ------ -------------------
2015-11-06 19:03:31 +08:00 2015-11-06 19:03:31
```
客户端查询：
```
SQL> alter session set nls_date_format='yyyy-dd-mm hh24:mi:ss';
会话已更改。
SQL> select sysdate,dbtimezone,current_date from dual;
SYSDATE             DBTIME CURRENT_DATE
------------------- ------ -------------------
2015-11-06 06:05:06 +08:00 2015-11-06 19:05:06
```
修改GI时区配置文件，TZ=Asia/Shanghai
```
[grid@oel6:/home/grid]$ sed -i 's/CST6CDT/Asia\/Shanghai/g' $ORACLE_HOME/crs/install/s_crsconfig_oel6_env.txt
```
重启crs，查看结果
client端：
```
SQL> alter session set nls_date_format='yyyy-dd-mm hh24:mi:ss';
Session altered.
SQL> select sysdate from dual;
SYSDATE
-------------------
2015-11-06 19:23:52
```
服务器端：
```
SQL> alter session set nls_date_format='yyyy-dd-mm hh24:mi:ss';
Session altered.
SQL> select sysdate from dual;
SYSDATE
-------------------
2015-11-06 19:24:30
```



Reference:   
[ID 1209444.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1209444.1)   
[如何诊断rac环境下sysdate 返回错误时间问题](https://blogs.oracle.com/Database4CN/entry/%E5%A6%82%E4%BD%95%E8%AF%8A%E6%96%ADrac%E7%8E%AF%E5%A2%83%E4%B8%8Bsysdate_%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E6%97%B6%E9%97%B4%E9%97%AE%E9%A2%98)


</br>
<b>EOF</b>
</br>

