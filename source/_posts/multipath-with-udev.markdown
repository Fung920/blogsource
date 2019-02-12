---
layout: post
title: "Multipath with udev in RHEL6"
date: 2015-06-15 21:51:56
comments: false
categories: oracle
tags: udev
keywords: multipath,udev
description: 
---
Multipath为linux自带多路径聚合软件，用于磁盘路径设备名称绑定，在6.x后期版本中，已经不支持multipath直接修改磁盘权限，需要通过udev进行设定。
<!--more-->
### 1.multipath安装
```
[root@orl6 ~]# yum install device-mapper-multipath -y
[root@orl6 ~]# /etc/init.d/multipathd start
Starting multipathd daemon: [  OK  ]
[root@orl6 ~]# chkconfig multipathd on
[root@orl6 ~]# multipath enable
Jun 15 22:09:36 | /etc/multipath.conf does not exist, blacklisting all devices.
Jun 15 22:09:36 | A sample multipath.conf file is located at
Jun 15 22:09:36 | /usr/share/doc/device-mapper-multipath-0.4.9/multipath.conf
Jun 15 22:09:36 | You can run /sbin/mpathconf to create or modify /etc/multipath.conf
[root@orl6 ~]# cp /usr/share/doc/device-mapper-multipath-0.4.9/multipath.conf /etc/multipath.conf
[root@orl6 ~]# grep -v ^# /etc/multipath.conf
defaults {
        user_friendly_names yes
}
```
### 2.配置multipath.conf
通过<code>scsi_id</code>命令找出磁盘wwid，并按照模板写入multipath.conf
```
[root@orl6 ~]# for i in a b c d;
> do 
> /sbin/scsi_id --whitelisted --replace-whitespace --device=/dev/sd$i
> done
1ATA_VBOX_HARDDISK_VBfe43d230-3b43fe42
1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b
14f504e46494c45526354355179652d663752632d6d755741
14f504e46494c45526354355179652d663752632d6d755741
```
可以看到，<code>/dev/sdc</code>和<code>/dev/sdd</code>是同一块磁盘，通过多路径聚合成一块，最后的配置文件如下：
```
[root@orl6 ~]# grep -v ^# /etc/multipath.conf
blacklist {
        #devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
        #devnode "^hd[a-z]"
        devnode "sda"            #排除本地硬盘
        #wwid 1ATA_VBOX_HARDDISK_VB2314099c-4ddf514a 
        #wwid 1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b
}
defaults {
        user_friendly_names no
        getuid_callout "/sbin/scsi_id --whitelisted --replace-whitespace --device=/dev/%n"
}
multipaths {
    multipath {
        wwid 1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b  #原有ASM磁盘
        alias mpath1
    }
    multipath {
        wwid 14f504e46494c45526354355179652d663752632d6d755741
        alias data_mpath2
    }
}
```
通过multipath路径即可发现聚合后的路径
```
[root@orl6 ~]# multipath -F
[root@orl6 ~]# multipath -v2
create: mpath1 (1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b) undef ATA,VBOX HARDDISK
size=20G features='0' hwhandler='0' wp=undef
`-+- policy='round-robin 0' prio=1 status=undef
  `- 3:0:0:0 sdb 8:16 undef ready running
create: data_mpath2 (14f504e46494c45526354355179652d663752632d6d755741) undef OPNFILER,VIRTUAL-DISK
size=15G features='0' hwhandler='0' wp=undef
|-+- policy='round-robin 0' prio=1 status=undef
| `- 5:0:0:0 sdc 8:32 undef ready running
`-+- policy='round-robin 0' prio=1 status=undef
  `- 4:0:0:0 sdd 8:48 undef ready running
```
Liunx自带device mapper的命令<code>dmsetup</code>:
```
[root@orl6 ~]# dmsetup ls
mpath1  (252:0)
data_mpath2     (252:1)
[root@orl6 ~]# dmsetup info
Name:              mpath1
State:             ACTIVE
Read Ahead:        256
Tables present:    LIVE
Open count:        22
Event number:      1
Major, minor:      252, 0
Number of targets: 1
UUID: mpath-1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b

Name:              data_mpath2
State:             ACTIVE
Read Ahead:        256
Tables present:    LIVE
Open count:        0
Event number:      0
Major, minor:      252, 1
Number of targets: 1
UUID: mpath-14f504e46494c45526354355179652d663752632d6d755741
```
### 3.通过udev设置磁盘权限
磁盘权限的设定在6后期版本中，multipath有自带的模板文件，如下：
```
[root@orl6 ~]# find / -name "12-dm-permissions.rules"
/usr/share/doc/device-mapper-1.02.79/12-dm-permissions.rules
[root@orl6 ~]# cat /usr/share/doc/device-mapper-1.02.79/12-dm-permissions.rules
--部分关于multipath的权限设定如下
# Copyright (C) 2009 Red Hat, Inc. All rights reserved.
#
# This file is part of LVM2.
# Udev rules for device-mapper devices.
#
# These rules set permissions for DM devices.
#
# This file is considered to be a template where users can put their
# own entries and then put a copy of it manually to a usual place with
# user-edited udev rules (usually /etc/udev/rules.d).
#
# There are some environment variables set that can be used:
#   DM_UDEV_RULES_VSN - DM udev rules version
#   DM_NAME - actual DM device's name
#   DM_UUID - UUID set for DM device (blank if not specified)
#   DM_SUSPENDED - suspended state of DM device (0 or 1)
#   DM_LV_NAME - logical volume name (not set if LVM device not present)
#   DM_VG_NAME - volume group name (not set if LVM device not present)
#   DM_LV_LAYER - logical volume layer (not set if LVM device not present)
# PLAIN DM DEVICES
#
# Set permissions for a DM device named 'my_device' exactly
# ENV{DM_NAME}=="my_device", OWNER:="root", GROUP:="root", MODE:="660"

# Set permissions for all DM devices having 'MY_UUID-' UUID prefix
# ENV{DM_UUID}=="MY_UUID-?*", OWNER:="root", GROUP:="root", MODE:="660"
# MULTIPATH DEVICES
#
# Set permissions for all multipath devices
# ENV{DM_UUID}=="mpath-?*", OWNER:="root", GROUP:="root", MODE:="660"

# Set permissions for first two partitions created on a multipath device (and detected by kpartx)
# ENV{DM_UUID}=="part[1-2]-mpath-?*", OWNER:="root", GROUP:="root", MODE:="660"
```
按照模板文件修改属主权限即可。
```
[root@orl6 ~]# cat /etc/udev/rules.d/12-asm-perssion.rules 
ENV{DM_NAME}=="mpath1",NAME="asmdisk1", OWNER:="grid", GROUP:="oinstall", MODE:="660"
ENV{DM_NAME}=="data_mpath2",NAME="asmdisk2", OWNER:="grid", GROUP:="oinstall", MODE:="660"
下面一段可以不加：
#[root@orl6 ~]# cat /etc/udev/rules.d/99-asm-rules.rules 
#KERNEL=="sd*", SUBSYSTEM=="block", ENV{DEVTYPE}=="disk", ENV{ID_SERIAL}=="1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b", OWNER="grid", GROUP="oinstall", MODE="0660"
#KERNEL=="sd*", SUBSYSTEM=="block", ENV{DEVTYPE}=="disk", ENV{ID_SERIAL}=="14f504e46494c45526354355179652d663752632d6d755741", OWNER="grid", GROUP="oinstall", MODE="0660"
#重启udev
[root@orl6 ~]# start_udev
Starting udev: [  OK  ]
[root@orl6 ~]# ll /dev/asm*
brw-rw----. 1 grid oinstall 252, 0 Jun 15 22:17 /dev/asmdisk1
brw-rw----. 1 grid oinstall 252, 1 Jun 15 22:17 /dev/asmdisk2
```
此时，grid用户通过asmca可发现可用磁盘。   
另外，udev自带的模板在[root@orl6 ~]# more /lib/udev/rules.d/50-udev-default.rules下。
### 4.EMC PowerPath使用udev
```
[root@db01 ~]# ls -l /dev/emc*
crw-r--r--. 1 root root  10,  56 Jun 10 14:56 /dev/emcpower
brw-rw----. 1 root disk 120,   0 Jun 10 14:56 /dev/emcpowera
brw-rw----. 1 root disk 120,  16 Jun 10 14:56 /dev/emcpowerb
brw-rw----. 1 root disk 120,  32 Jun 10 14:56 /dev/emcpowerc
brw-rw----. 1 root disk 120,  48 Jun 10 14:56 /dev/emcpowerd
brw-rw----. 1 root disk 120,  64 Jun 10 14:56 /dev/emcpowere
brw-rw----. 1 root disk 120,  80 Jun 10 14:56 /dev/emcpowerf
brw-rw----. 1 root disk 120,  96 Jun 10 14:56 /dev/emcpowerg
brw-rw----. 1 root disk 120, 112 Jun 10 14:56 /dev/emcpowerh
brw-rw----. 1 root disk 120, 128 Jun 10 14:56 /dev/emcpoweri
[root@db01 rules.d]# cat 99-oracle-asmdevices.rules
SUBSYSTEM=="block", KERNEL=="emcpowera", GROUP="dba", OWNER="grid", MODE="0660"
SUBSYSTEM=="block", KERNEL=="emcpowerb", GROUP="dba", OWNER="grid", MODE="0660"
SUBSYSTEM=="block", KERNEL=="emcpowerc", GROUP="dba", OWNER="grid", MODE="0660"
SUBSYSTEM=="block", KERNEL=="emcpowerd", GROUP="dba", OWNER="grid", MODE="0660"
...
[root@db01 rules.d]# start_udev
[root@db01 app]# ll /dev/emc*
crw-rw---- 1 root root  10,  56 6月  10 15:30 /dev/emcpower
brw-rw---- 1 grid dba  120,   0 6月  10 15:34 /dev/emcpowera
brw-rw---- 1 grid dba  120,  16 6月  10 15:34 /dev/emcpowerb
brw-rw---- 1 grid dba  120,  32 6月  10 15:34 /dev/emcpowerc
brw-rw---- 1 grid dba  120,  48 6月  10 15:34 /dev/emcpowerd
brw-rw---- 1 grid dba  120,  64 6月  10 15:34 /dev/emcpowere
brw-rw---- 1 grid dba  120,  80 6月  10 15:34 /dev/emcpowerf
brw-rw---- 1 grid dba  120,  96 6月  10 15:34 /dev/emcpowerg
brw-rw---- 1 grid dba  120, 112 6月  10 15:34 /dev/emcpowerh
brw-rw---- 1 grid dba  120, 128 6月  10 15:34 /dev/emcpoweri
```
### 5.Multipathd指令
```
--检查设备
[root@orl6 ~]# multipathd show devices
available block devices:
    sda devnode blacklisted, unmonitored
    sdb devnode whitelisted, monitored
    sdc devnode whitelisted, monitored
    sdd devnode whitelisted, monitored
    sr0 devnode blacklisted, unmonitored
    sr1 devnode blacklisted, unmonitored
    dm-0 devnode blacklisted, unmonitored
...
--检查映射情况
[root@orl6 ~]# multipathd show maps
name        sysfs uuid                                             
mpath1      dm-0  1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b           
data_mpath2 dm-1  14f504e46494c45526354355179652d663752632d6d755741
[root@orl6 ~]# multipathd show topology
#此命令等同于<code>multipath -ll</code>
mpath1 (1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b) dm-0 ATA,VBOX HARDDISK
size=20G features='0' hwhandler='0' wp=rw
`-+- policy='round-robin 0' prio=1 status=active
  `- 3:0:0:0 sdb 8:16 active ready running
data_mpath2 (14f504e46494c45526354355179652d663752632d6d755741) dm-1 OPNFILER,VIRTUAL-DISK
size=15G features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 5:0:0:0 sdc 8:32 active ready running
`-+- policy='round-robin 0' prio=1 status=active
  `- 4:0:0:0 sdd 8:48 active ready running
```
更加详细的命令可以参照man，<code>multipathd -k</code>为交互模式。
### 6.Multipath相关配置文件
```
[root@orl6 ~]# cat /etc/multipath/bindings 
# Multipath bindings, Version : 1.0
# NOTE: this file is automatically maintained by the multipath program.
# You should not need to edit this file in normal circumstances.
#
# Format:
# alias wwid
#
mpatha 1ATA     VBOX HARDDISK                           VBfe43d230-3b43fe42 
mpathb 1ATA     VBOX HARDDISK                           VB9b70e1bb-9ea5e71b 
mpathc 14f504e46494c45526354355179652d663752632d6d755741
[root@orl6 ~]# cat /etc/multipath/wwids 
# Multipath wwids, Version : 1.0
# NOTE: This file is automatically maintained by multipath and multipathd.
# You should not need to edit this file in normal circumstances.
#
# Valid WWIDs:
/1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b/
/14f504e46494c45526354355179652d663752632d6d755741/
[root@orl6 ~]# grep -v ^# /etc/multipath.conf |grep -v ^$
blacklist {
        #devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
        #devnode "^hd[a-z]"
        devnode "sda"
        #wwid 1ATA_VBOX_HARDDISK_VB2314099c-4ddf514a 
        #wwid 1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b
}
defaults {
        user_friendly_names no
        getuid_callout "/sbin/scsi_id --whitelisted --replace-whitespace --device=/dev/%n"
}
multipaths {
    multipath {
        wwid 1ATA_VBOX_HARDDISK_VB9b70e1bb-9ea5e71b
        alias mpath1
    }
    multipath {
        wwid 14f504e46494c45526354355179652d663752632d6d755741
        alias data_mpath2
    }
}
```

### 7. RHEL7.4 ASM udev加盘
```
/usr/bin/scsi-rescan
multipath -ll
vi /etc/udev/rule.d/99-oracle-asmdevices.rules
--restart udev
/sbin/udevadm trigger --type=devices --action=change
```

* Find the UUID of the disk

```
udevadm info --query=all --name=/dev/mapper/mpathx | grep -i DM_UUID
```

* Create udev Rules

```
vi /etc/udev/rules.d/96-asm.rules
ACTION=="add|change", ENV{DM_UUID}=="mpath-[DM_UUID]", SYMLINK+="udev-asmdisk1", GROUP="oinstall", OWNER="grid", MODE="0660"
```

* Reload udev Rules

```
udevadm control --reload-rules
udevadm trigger --type=devices --action=change
```

* Verify the disks with sg_inq command

```
# su - grid
$ sg_inq /dev/mapper/mpathx
$ sg_inq /dev/dm-x
```


Reference:   
[ID 1538626.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1538626.1)   
[ID 1521757.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1521757.1)   

</br>
<b>EOF</b>
</br>

