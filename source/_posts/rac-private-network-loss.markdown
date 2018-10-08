---
layout: post
title: "心跳丢失引起RAC节点不断重启"
date: 2014-08-26 17:25:21
comments: false
categories: oracle
tags: rac
keywords: ora-00481
description: 心跳丢失引起RAC节点不断重启
---
客户双节点RAC在8月22号晚上节点2异常终止，随即自动重新启动，但启动失败，25号下午17点多手动启动节点2上的Cluster，结果在26号客户巡检发现节点2在不停的重启。
<!--more-->
环境：11.2.0.3 + Linux 64bit 双节点 RAC  
在8月22号开始，节点2首次出现
```
Fri Aug 22 19:26:27 2014
PMON (ospid: 13912): terminating the instance due to error 481
Instance terminated by PMON, pid = 13912
```
节点2开始自动重启，重启过程中出现
```
NOTE: client +ASM2:+ASM registered, osid 20296, mbr 0x0
WARNING: failed to online diskgroup resource ora.DATA.dg (unable to communicate with CRSD/OHASD)
WARNING: failed to online diskgroup resource ora.RECOVERY.dg (unable to communicate with CRSD/OHASD)
```
再次报错
```
Fri Aug 22 19:41:42 2014
PMON (ospid: 20218): terminating the instance due to error 481
Instance terminated by PMON, pid = 20218
```
ora-00481错误信息如下：
```
ORA-00481:LMON process terminated with error
Cause:	 The global enqueue service monitor process died
Action:	 Warm start instance
```
LMON: Global Enqueue Service Monitor。LMON用于监控整个集群的global enqueues和resources， 而且会执行global enqueue recovery。实例异常终止后，会由LMON来进行GCS内存方面的处理。当一个实例加入或者离开集群后，LMON会对lock和resource进行reconfiguration.另外LMON会在不同的实例间进行通讯检查，如果发现对方通讯超时，就会发出节点eviction。  
22号相关日志：
```
--alter_db2.log
Fri Aug 22 19:26:25 2014  --第一次错误发生时间为22号19:26
NOTE: ASMB terminating
Errors in file /opt/u01/app/oracle/diag/rdbms/dbrac/dbrac2/trace/dbrac2_asmb_26030.trc:
ORA-15064: ? ASM ??????
ORA-03113: ?????????
?? ID: 
?? ID: 82 ???: 35
Errors in file /opt/u01/app/oracle/diag/rdbms/dbrac/dbrac2/trace/dbrac2_asmb_26030.trc:
ORA-15064: ? ASM ??????
ORA-03113: ?????????
?? ID: 
?? ID: 82 ???: 35
ASMB (ospid: 26030): terminating the instance due to error 15064
Instance terminated by ASMB, pid = 26030
Fri Aug 22 19:28:02 2014
Starting ORACLE instance (normal) --开始自动重启Cluster
```
#####alternode2.log:
```
2014-08-22 19:26:16.065   --时间为22号19:26
[cssd(13669)]CRS-1612:50% 的超时时间间隔内缺少与节点 dbserver_node1 (1) 的网络通信。将在 14.140 秒后从集群中删除此节点
2014-08-22 19:26:23.079
[cssd(13669)]CRS-1611:75% 的超时时间间隔内缺少与节点 dbserver_node1 (1) 的网络通信。将在 7.130 秒后从集群中删除此节点
```
#####ocssd.log：
```
2014-08-22 19:26:16.065: [    CSSD][1113200960]clssnmPollingThread: node dbserver_node1 (1) at 50% heartbeat fatal, removal in 14.140 seconds 
--Heartbeat不通
2014-08-22 19:26:16.065: [    CSSD][1113200960]clssnmPollingThread: node dbserver_node1 (1) is impending reconfig, flag 2491406, misstime 15860
2014-08-22 19:26:16.065: [    CSSD][1113200960]clssnmPollingThread: local diskTimeout set to 27000 ms, remote disk timeout set to 27000, impending reconfig status(1)
2014-08-22 19:26:16.066: [    CSSD][1106893120]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030317, wrtcnt, 27424336, LATS 576293252, lastSeqNo 25251022, uniqueness 1399539197, timestamp 1408706775/576374502
2014-08-22 19:26:17.005: [    CSSD][1114777920]clssnmSendingThread: sending status msg to all nodes
2014-08-22 19:26:17.005: [    CSSD][1114777920]clssnmSendingThread: sent 4 status msgs to all nodes
2014-08-22 19:26:17.068: [    CSSD][1106893120]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030317, wrtcnt, 27424342, LATS 576294252, lastSeqNo 27424336, uniqueness 1399539197, timestamp 1408706776/576375512 
--心跳出问题，内部通信有问题
```
26号部分日志，情况跟22号一样：
#####db alter_node2.log
```
Tue Aug 26 15:15:21 2014  --时间在15:15分左右
NOTE: ASMB terminating
Errors in file /opt/u01/app/oracle/diag/rdbms/dbrac/dbrac2/trace/dbrac2_asmb_27935.trc:
ORA-15064: ? ASM ??????
ORA-03113: ?????????
?? ID: 
?? ID: 4 ???: 5
Errors in file /opt/u01/app/oracle/diag/rdbms/dbrac/dbrac2/trace/dbrac2_asmb_27935.trc:
ORA-15064: ? ASM ??????
ORA-03113: ?????????
?? ID: 
?? ID: 4 ???: 5
ASMB (ospid: 27935): terminating the instance due to error 15064
Instance terminated by ASMB, pid = 27935
Tue Aug 26 15:17:00 2014
Starting ORACLE instance (normal)
```
#####ocssd.log
```
2014-08-26 15:15:11.914: [    CSSD][1100093760]clssnmPollingThread: node dbserver_node1 (1) at 50% heartbeat fatal, removal in 14.070 seconds
2014-08-26 15:15:11.914: [    CSSD][1100093760]clssnmPollingThread: node dbserver_node1 (1) is impending reconfig, flag 2491406, misstime 15930
2014-08-26 15:15:11.914: [    CSSD][1100093760]clssnmPollingThread: local diskTimeout set to 27000 ms, remote disk timeout set to 27000, impending reconfig status(1)
2014-08-26 15:15:11.915: [    CSSD][1088473408]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415629, LATS 906787192, lastSeqNo 0, uniqueness 1399539197, timestamp 1409037311/906871422
2014-08-26 15:15:11.915: [    CSSD][1108830528]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415631, LATS 906787192, lastSeqNo 28412102, uniqueness 1399539197, timestamp 1409037311/906871432
2014-08-26 15:15:12.917: [    CSSD][1088473408]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415635, LATS 906788192, lastSeqNo 28415629, uniqueness 1399539197, timestamp 1409037312/906872442
2014-08-26 15:15:12.917: [    CSSD][1108830528]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415637, LATS 906788192, lastSeqNo 28415631, uniqueness 1399539197, timestamp 1409037312/906872452
2014-08-26 15:15:13.920: [    CSSD][1088473408]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415641, LATS 906789202, lastSeqNo 28415635, uniqueness 1399539197, timestamp 1409037313/906873442
2014-08-26 15:15:13.920: [    CSSD][1108830528]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415643, LATS 906789202, lastSeqNo 28415637, uniqueness 1399539197, timestamp 1409037313/906873452
2014-08-26 15:15:14.921: [    CSSD][1088473408]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415647, LATS 906790202, lastSeqNo 28415641, uniqueness 1399539197, timestamp 1409037314/906874442
2014-08-26 15:15:14.922: [    CSSD][1108830528]clssnmvDHBValidateNCopy: node 1, dbserver_node1, has a disk HB, but no network HB, DHB has rcfg 295030399, wrtcnt, 28415649, LATS 906790202, lastSeqNo 28415643, uniqueness 1399539197, timestamp 1409037314/906874452
```
最后ping 大包，确定心跳网络有问题，丢包率达到15%。
```
ping -s 1500 node2-priv
```
![ping1](/images/rac_brain_split1.png)  
![ping2](/images/rac_brain_split2.png)