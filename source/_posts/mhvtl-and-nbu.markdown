---
layout: post
title: "MHVTL and NBU"
date: 2015-07-20 11:19:54
comments: false
categories: linux
tags: vtl
keywords: vtl,nbu
description: how to setup mhvtl in rhel6.5 and using nbu
---
MHVTL 为开源虚拟带库软件，下载地址为：[MHVTL](https://sites.google.com/site/linuxvtl2)，搭配上gui可模拟多款厂商带库，我唯一管理过的带库为HP MSL6000，而备份软件则为Netbackup。本文以这两种产品为例，对一些简单的配置进行说明。
<!--more-->
### 1、安装MHVTL
OS版本为RHEL6.5，以最小安装，通过配置yum本地源安装其他所需的包，其中，有三个rpm包需要自行下载，可从[http://rpmfind.net/linux/rpm2html/search.php?query=lzo](http://rpmfind.net/linux/rpm2html/search.php?query=lzo)这里搜索对应版本或者在本站下载：[lzo-2.03-3.1.el6_5.1.x86_64.rpm](/download/mhvtl/rpm/lzo-2.03-3.1.el6_5.1.x86_64.rpm),[lzo-devel-2.03-3.1.el6_5.1.x86_64.rpm](/download/mhvtl/rpm/lzo-devel-2.03-3.1.el6_5.1.x86_64.rpm),[lzo-minilzo-2.03-3.1.el6_5.1.x86_64.rpm](/download/mhvtl/rpm/lzo-minilzo-2.03-3.1.el6_5.1.x86_64.rpm)
#### 1.1关闭firewall和selinux
```
#关闭防火墙
[root@mhvtl ~]# /etc/init.d/iptables stop
iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
iptables: Flushing firewall rules:                         [  OK  ]
iptables: Unloading modules:                               [  OK  ]
[root@mhvtl ~]# /etc/init.d/ip6tables stop
ip6tables: Setting chains to policy ACCEPT: filter         [  OK  ]
ip6tables: Flushing firewall rules:                        [  OK  ]
ip6tables: Unloading modules:                              [  OK  ]
[root@mhvtl ~]# chkconfig iptables off
[root@mhvtl ~]# chkconfig ip6tables off
#关闭selinux
[root@mhvtl ~]# sed -i '/^SELINUX/s/enforcing/disabled/g' /etc/selinux/config
[root@mhvtl ~]# cat /etc/selinux/config 
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted 
[root@mhvtl ~]# shutdown -ry 0
```
#### 1.2配置yum本地源
```
[Server] 
name=Server 
baseurl=file:///mnt/Server 
enabled=1 
gpgcheck=0
```
挂载光盘，安装所需软件包
```
[root@mhvtl ~]# mount /dev/sr1 /mnt
mount: block device /dev/sr1 is write-protected, mounting read-only
[root@mhvtl ~]# yum -y install zlib-devel mtx mt-st lsscsi kernel-devel kernel-headers sg3_utils gcc perl unzip
[root@mhvtl worktmp]# ls -ltr
total 708
-rw-r--r--. 1 root root 283656 Jul 16 14:12 mhvtl-2015-04-14.tgz
-rw-r--r--. 1 root root  12876 Jul 16 15:14 lzo-minilzo-2.03-3.1.el6_5.1.x86_64.rpm
-rw-r--r--. 1 root root  31784 Jul 16 15:17 lzo-devel-2.03-3.1.el6_5.1.x86_64.rpm
-rw-r--r--. 1 root root  56308 Jul 16 15:20 lzo-2.03-3.1.el6_5.1.x86_64.rpm
-rw-r--r--. 1 root root 329355 Jul 16 15:54 mhvtl-gui-master.zip
[root@mhvtl worktmp]# rpm -ivh *.rpm
warning: lzo-2.03-3.1.el6_5.1.x86_64.rpm: Header V3 RSA/SHA1 Signature, key ID c105b9de: NOKEY
Preparing...                ########################################### [100%]
   1:lzo-minilzo            ########################################### [ 33%]
   2:lzo                    ########################################### [ 67%]
   3:lzo-devel              ########################################### [100%]
[root@mhvtl worktmp]# 
```
#### 1.3编译安装MHVTL
创建vtl相关用户：
```
[root@mhvtl mhvtl-1.5]# groupadd vtl -g 600
[root@mhvtl mhvtl-1.5]# useradd -g vtl -u 600 vtl
[root@mhvtl mhvtl-1.5]# echo "oracle" | passwd vtl --stdin > /dev/null 2>&1
```
编译安装vtl
```
[root@mhvtl mhvtl-1.5]# pwd
/worktmp/mhvtl-1.5
[root@mhvtl mhvtl-1.5]# cd kernel/
[root@mhvtl kernel]# make
[root@mhvtl kernel]# make install
[root@mhvtl kernel]#
[root@mhvtl kernel]# cd ../
[root@mhvtl mhvtl-1.5]# make 
[root@mhvtl mhvtl-1.5]# make install
```
启动服务，并且查看虚拟磁带：
```
[root@mhvtl mhvtl-1.5]# mkdir -p /opt/mhvtl
[root@mhvtl mhvtl-1.5]# chown vtl:vtl /opt/mhvtl/
[root@mhvtl mhvtl-1.5]# /etc/init.d/mhvtl start
vtllibrary process PID is 3722
vtllibrary process PID is 3725
[root@mhvtl mhvtl-1.5]# ll /opt/mhvtl/
total 288
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 CLN101L4
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 CLN102L5
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 CLN303TA
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 E01001L4
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 E01002L4
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 E01003L4
drwxrwx--- 2 vtl vtl 4096 Jul 20 12:39 E01004L4
[root@mhvtl mhvtl-1.5]# lsscsi -g
[0:0:0:0]    cd/dvd  VBOX     CD-ROM           1.0   /dev/sr0   /dev/sg0
[0:0:1:0]    cd/dvd  VBOX     CD-ROM           1.0   /dev/sr1   /dev/sg1
[1:0:0:0]    cd/dvd  VBOX     CD-ROM           1.0   /dev/sr2   /dev/sg2
[2:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda   /dev/sg3
[3:0:0:0]    mediumx STK      L700             0105  /dev/sch0  /dev/sg12
[3:0:1:0]    tape    IBM      ULT3580-TD5      0105  /dev/st0   /dev/sg4
[3:0:2:0]    tape    IBM      ULT3580-TD5      0105  /dev/st1   /dev/sg5
[3:0:3:0]    tape    IBM      ULT3580-TD4      0105  /dev/st2   /dev/sg6
[3:0:4:0]    tape    IBM      ULT3580-TD4      0105  /dev/st3   /dev/sg7
[3:0:8:0]    mediumx STK      L80              0105  /dev/sch1  /dev/sg13
[3:0:9:0]    tape    STK      T10000B          0105  /dev/st4   /dev/sg8
[3:0:10:0]   tape    STK      T10000B          0105  /dev/st5   /dev/sg9
[3:0:11:0]   tape    STK      T10000B          0105  /dev/st6   /dev/sg10
[3:0:12:0]   tape    STK      T10000B          0105  /dev/st7   /dev/sg11
```
从lssci可以看到，默认配置是两台带库，STK的L700和STK的L80。我们能够通过gui进行添加和删除带库，但最后一个带库不能被删除。
### 2、GUI配置
GUI下载及安装说明：[https://github.com/walterfrs/mhvtl-gui](https://github.com/walterfrs/mhvtl-gui)，也可以从本站下载：[MHVL-GUI](/download/mhvtl/gui/mhvtl-gui-master.zip)。
#### 2.1. 设置http
```
[root@mhvtl mhvtl-gui-master]# pwd
/worktmp/mhvtl-gui-master
[root@mhvtl mhvtl-gui-master]# ls -l
total 64
-rw-r--r-- 1 root root  1150 Nov 28  2012 favicon.ico
-rwxr-xr-x 1 root root   750 Nov 28  2012 go.php
drwxr-xr-x 3 root root 12288 Nov 28  2012 html
-rw-r--r-- 1 root root   343 Nov 28  2012 index.php
-rw-r--r-- 1 root root 18006 Nov 28  2012 LICENSE
-rw-r--r-- 1 root root  2961 Nov 28  2012 login.php
-rw-r--r-- 1 root root  2379 Nov 28  2012 mhvtl.cfg.db
-rw-r--r-- 1 root root  2931 Nov 28  2012 README
drwxr-xr-x 2 root root  4096 Nov 28  2012 scripts
-rw-r--r-- 1 root root    14 Nov 28  2012 version
[root@mhvtl mhvtl-gui-master]# yum install httpd -y
[root@mhvtl mhvtl-gui-master]# chkconfig httpd on
[root@mhvtl mhvtl-gui-master]# cp -r * /var/www/html/
```
#### 2.2.设置GUI
```
[root@mhvtl mhvtl-gui-master]# echo "apache ALL=(ALL) NOPASSWD: ALL" >>/etc/sudoers
```
屏蔽掉/et/sudoers中的"Defaults requiretty"
```
[root@mhvtl mhvtl-gui-master]# grep 'Defaults    requiretty' /etc/sudoers
Defaults    requiretty
[root@mhvtl mhvtl-gui-master]# sed -i '/^Defaults    requiretty/s/^/#/g' /etc/sudoers
[root@mhvtl mhvtl-gui-master]# grep 'Defaults    requiretty' /etc/sudoers
#Defaults    requiretty
```
安装所需rpm包：
```
[root@mhvtl mhvtl-gui-master]# yum install php sysstat git iscsi-initiator-utils* scsi-target-utils -y
[root@mhvtl mhvtl-gui-master]# yum groupinstall "Desktop" -y
```
配置html网站的alias,将如下内容添加到<code>/etc/httpd/conf/httpd.conf</code>
```
Alias /mhvtl "/var/www/html/mhvtl"
<Directory "/var/www/html/mhvtl">
   Options None
   AllowOverride None
   Order allow,deny
   Allow from all
</Directory>
```
创建所需目录
```
[root@mhvtl mhvtl-gui-master]# mkdir -p /var/www/html/mhvtl
```
OK，httpd服务，并测试连接：
```
[root@mhvtl mhvtl-gui-master]# /etc/init.d/httpd restart
Stopping httpd:                                            [FAILED]
Starting httpd: httpd: apr_sockaddr_info_get() failed for mhvtl
httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1 for ServerName
                                                           [  OK  ]
```
上述报错是忘记在<code>/etc/hosts</code>添加主机名及IP了，添加后重启httpd服务正常。   
![mhvtl-gui1](/images/mhvtl-gui1.png)
默认密码为mhvtl   
![mhvtl-gui2](/images/mhvtl-gui2.png)
图中红框处为默认两台带库，下面开始配置自己的带库。   
### 3、配置带库及iscsi target
点击setup=>Remove,选中要删除的带库，同时删除磁带：   
![rm-lib1](/images/rm-lib1.png)
完成后，因为最后一个带库不能被删除，因此，添加一个我们所需的带库，然后再删除默认带库。
点击setup=>Add=>Standard=>Next，在brand选中HP，最后我的配置如下，只选了LTO3的磁带   
![add-lib1](/images/add-lib1.png)
然后可以将默认磁带库删除了。同时，在vtl主机端可以看到设备类型：
```
[root@mhvtl html]# cat /etc/mhvtl/device.conf

VERSION: 5

# VPD page format:
# <page #> <Length> <x> <x+1>... <x+n>
# NAA format is an 8 hex byte value seperated by ':'
# Note: NAA is part of inquiry VPD 0x83
#
# Each 'record' is separated by one (or more) blank lines.
# Each 'record' starts at column 1
# Serial num max len is 10.
# Compression: factor X enabled 0|1
#     Where X is zlib compression factor        1 = Fastest compression
#                                               9 = Best compression
#     enabled 0 == off, 1 == on
#
# fifo: /var/tmp/mhvtl
# If enabled, data must be read from fifo, otherwise daemon will block
# trying to write.
# e.g. cat /var/tmp/mhvtl (in another terminal)

Library: 50 CHANNEL: 1 TARGET: 00 LUN: 00
 Vendor identification: HP
 Product identification: MSL6000 Series	--带库型号
 Product revision level: 2.00
 Unit serial number: 80000050
 NAA: 50:11:22:33:ab:1:00:00
 Home directory: /opt/mhvtl
 Backoff: 400

Drive: 51 CHANNEL: 1 TARGET: 00 LUN: 01
 Library ID: 50 Slot: 01
 Vendor identification: HP
 Product identification: Ultrium 3-SCSI	--driver型号
 Product revision level: N11G
 Unit serial number: 80000051
 NAA: 50:11:22:33:ab:1:00:01
 Compression: factor 1 enabled 1
 Compression type: lzo
 Backoff: 400
...
```
最后状态如图：   
![add-lib2](/images/add-lib2.png)
可以看到，iscsi target是offline的状态，下面开始配置iscsi target:   
点击iscsi，将打叉的服务全部安装或enable   
![iscsi-setup1](/images/iscsi-setup1.png)
最后状态如图：   
![iscsi-setup2](/images/iscsi-setup2.png)
配置iscsi target，点击new，然后点击create，名字可以自己取，如下图：   
![iscsi-setup3](/images/iscsi-setup3.png)
至此，gui界面配置完毕，在vtl主机端，可以看到iscsi target：
```
[root@mhvtl html]# tgtadm --lld iscsi --op show --mode target
Target 1: iqn.1994-05.com.redhat:mhvtl:mhvtl:tgt:1
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
    Account information:
    ACL information:
```
最后，尚须手动添加磁带机及drivers信息到iscsi target中：
```
[root@mhvtl html]# lsscsi -g
[0:0:0:0]    cd/dvd  VBOX     CD-ROM           1.0   /dev/sr0   /dev/sg0
[0:0:1:0]    cd/dvd  VBOX     CD-ROM           1.0   /dev/sr1   /dev/sg1
[1:0:0:0]    cd/dvd  VBOX     CD-ROM           1.0   /dev/sr2   /dev/sg2
[2:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda   /dev/sg3
[3:1:0:0]    mediumx HP       MSL6000 Series   2.00  /dev/sch0  /dev/sg8
[3:1:0:1]    tape    HP       Ultrium 3-SCSI   N11G  /dev/st0   /dev/sg4
[3:1:0:2]    tape    HP       Ultrium 3-SCSI   N11G  /dev/st1   /dev/sg5
[3:1:0:3]    tape    HP       Ultrium 3-SCSI   N11G  /dev/st2   /dev/sg6
[3:1:0:4]    tape    HP       Ultrium 3-SCSI   N11G  /dev/st3   /dev/sg7
```
将标记有HP的设备信息写入<code>/etc/tgt/targets.conf</code>，如下：
```
<target iqn.1994-05.com.redhat:mhvtl:mhvtl:tgt:1>
backing-store /dev/sg8
backing-store /dev/sg4
backing-store /dev/sg5
backing-store /dev/sg6
backing-store /dev/sg7
device-type pt
bs-type sg
</target>
initiator-address ALL
```
重启iscsi target服务，并且验证
```
[root@mhvtl html]# /etc/init.d/tgtd restart
Stopping SCSI target daemon:                               [  OK  ]
Starting SCSI target daemon:                               [  OK  ]
[root@mhvtl html]# chkconfig tgtd on
[root@mhvtl html]# tgtadm --lld iscsi --op show --mode target
Target 1: iqn.1994-05.com.redhat:mhvtl:mhvtl:tgt:1
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: passthrough
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: sg
            Backing store path: /dev/sg4
            Backing store flags: 
        LUN: 2
            Type: passthrough
            SCSI ID: IET     00010002
            SCSI SN: beaf12
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: sg
            Backing store path: /dev/sg5
            Backing store flags: 
        LUN: 3
            Type: passthrough
            SCSI ID: IET     00010003
            SCSI SN: beaf13
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: sg
            Backing store path: /dev/sg6
            Backing store flags: 
        LUN: 4
            Type: passthrough
            SCSI ID: IET     00010004
            SCSI SN: beaf14
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: sg
            Backing store path: /dev/sg7
            Backing store flags: 
        LUN: 5
            Type: passthrough
            SCSI ID: IET     00010005
            SCSI SN: beaf15
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            Backing store type: sg
            Backing store path: /dev/sg8
            Backing store flags: 
    Account information:
    ACL information:
        ALL
```
### 4、windows server端配置iscsi
本例中，NBU服务器为windows2008r2，配置好主机名及etc/hosts后，设置发现iscsi，如下图，在发现门户中输入虚拟带库IP地址   
![win-iscsi1](/images/win-iscsi1.png)
然后在目标中连接目标iscsi target。   
![win-iscsi2](/images/win-iscsi2.png)
此时，在设备管理器中能够发现磁带库驱动及磁带：   
![drivers1](/images/drivers1.png)
### 5、安装NBU Master Server
点击安装文件中的Browser，有安装前预览及其他任务，针对于7.5以上版本的NBU，建议内存大于等于8GB，且安装的时候不要用terminal登陆。   
![nbu-inst1](/images/nbu-inst1.png)
下面几步可以一直下一步，在选择安装类型的时候可以选择typical或者custom，这里选择typical即可，custom可以指定各个进程的端口号及安装路径。   
![nbu-inst2](/images/nbu-inst2.png)
输入license，点击master server安装   
![nbu-inst3](/images/nbu-inst3.png)
等待NBU安装完成。
### 6、配置NBU机械臂及driver等设备
启动NBU Admin console，点击getting start开始配置drivers和robots：   
![nbu-conf1](/images/nbu-conf1.png)
直接下一步，扫描完会发现VTL中配置的4个drivers和一个robot   
![nbu-conf2](/images/nbu-conf2.png)
在driver的配置中，有可能会发生drivers出现在standalone里面，需要我们手动将四个drivers拖拽到robot 0里面：   
![nbu-conf3](/images/nbu-conf3.png)
完成后，如图：   
![nbu-conf4](/images/nbu-conf4.png)
配置完storage devices后，开始配置volume：   
![nbu-conf5](/images/nbu-conf5.png)
在下面这个页面中，选中TLD，NBU会扫描带库中所有磁带，并添加到NBU的inventory中：   
![nbu-conf6](/images/nbu-conf6.png)   
![nbu-conf7](/images/nbu-conf7.png)   
![nbu-conf8](/images/nbu-conf8.png)
catalog的policy及backup policy在后面手动配置。至此，NBU基于磁带的主要配置已经完成。
### 7、本地磁盘配置
在某些环境下，有些master server挂载了一些存储，我们可以直接配置NBU，将数据备份在local disk：   
![nbu-conf9](/images/nbu-conf9.png)
配置完成后，在policy中，可以选择基于disk的storage units，将备份放在本地硬盘。

</br>
<b>EOF</b>
</br>
