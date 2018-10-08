---
layout: post
title: "AIX基本操作"
date: 2014-05-16 16:18:33 +0800
comments: false
categories: aix
tags: aix
keywords: aix,basic command
description: some aix basic operations
---
<h3>1.change swap size</h3>
<!--more-->
```
--check current swap size
# lsps -a
Page Space      Physical Volume   Volume Group Size %Used Active  Auto  Type Chksum
hd6             hdisk0            rootvg         512MB     2   yes   yes    lv     0
--check the memory size in the mechine
# lsattr -Elmem0
ent_mem_cap         I/O memory entitlement in Kbytes           False
goodsize       7648 Amount of usable physical memory in Mbytes False
mem_exp_factor      Memory expansion factor                    False
size           7648 Total amount of physical memory in Mbytes  False
var_mem_weight      Variable memory capacity weight            False
--check rootvg PP size
# lsvg rootvg
VOLUME GROUP:       rootvg                   VG IDENTIFIER:  00c868bf00004c000000000000848812
VG STATE:           active                   PP SIZE:        128 megabyte(s)
VG PERMISSION:      read/write               TOTAL PPs:      1345 (172160 megabytes)
MAX LVs:            256                      FREE PPs:       1147 (146816 megabytes)
LVs:                12                       USED PPs:       198 (25344 megabytes)
OPEN LVs:           11                       QUORUM:         2 (Enabled)
TOTAL PVs:          2                        VG DESCRIPTORS: 3
STALE PVs:          0                        STALE PPs:      0
ACTIVE PVs:         1                        AUTO ON:        yes
MAX PPs per VG:     32512
MAX PPs per PV:     1016                     MAX PVs:        32
LTG size (Dynamic): 128 kilobyte(s)          AUTO SYNC:      no
HOT SPARE:          no                       BB POLICY:      relocatable
PV RESTRICTION:     none
--increase swap to 8GB
# chps -s 64 hd6
lquerypv: Warning, physical volume hdisk2 is excluded since it may be
        either missing or removed.
# lsps -a
Page Space      Physical Volume   Volume Group Size %Used Active  Auto  Type Chksum
hd6             hdisk0            rootvg        8704MB     1   yes   yes    lv     0
```
<h3>2.create file system</h3>
```
--check vg
# lsvg
rootvg
# lsvg -p rootvg
rootvg:
PV_NAME           PV STATE          TOTAL PPs   FREE PPs    FREE DISTRIBUTION
hdisk0            active            546         136         108..18..00..00..10
#
--check pv
# lspv
hdisk0          00c868bfc7fd26e6                    rootvg          active
# lsdev -Cc disk
hdisk0 Available 04-08-00-8,0 16 Bit LVD SCSI Disk Drive
hdisk1 Defined   02-08-01     MPIO Other DS4K Array Disk
hdisk2 Defined   02-08-01     MPIO Other DS4K Array Disk
hdisk3 Defined   02-08-01     MPIO Other DS4K Array Disk
# lspv hdisk0
PHYSICAL VOLUME:    hdisk0                   VOLUME GROUP:     rootvg
PV IDENTIFIER:      00c868bfc7fd26e6 VG IDENTIFIER     00c868bf00004c000000000000848812
PV STATE:           active
STALE PARTITIONS:   0                        ALLOCATABLE:      yes
PP SIZE:            128 megabyte(s)          LOGICAL VOLUMES:  12
TOTAL PPs:          546 (69888 megabytes)    VG DESCRIPTORS:   2
FREE PPs:           348 (44544 megabytes)    HOT SPARE:        no
USED PPs:           198 (25344 megabytes)    MAX REQUEST:      256 kilobytes
FREE DISTRIBUTION:  109..21..00..109..109
USED DISTRIBUTION:  01..88..109..00..00
MIRROR POOL:        None
--create vg
# mkvg -y oravg -s 128M hdisk1
--create lv
# mklv -t jfs2 -y oralv rootvg 24 hdisk0
oralv
--create file system
# mkdir /oracle
# crfs -v jfs2 -d oralv -A yes -m /oracle
File system created successfully.
3145428 kilobytes total disk space.
New File System size is 6291456
# mount /oracle
# df -g
Filesystem    GB blocks      Free %Used    Iused %Iused Mounted on
/dev/hd4           0.25      0.07   74%    10612    40% /
/dev/hd2           2.25      0.04   99%    49260    80% /usr
/dev/hd9var        0.50      0.19   63%     8223    16% /var
/dev/hd3          10.00      9.53    5%      748     1% /tmp
/dev/hd1           0.12      0.12    1%        5     1% /home
/dev/hd11admin      0.12      0.12    1%        5     1% /admin
/proc                 -         -    -         -     -  /proc
/dev/hd10opt       0.50      0.26   48%     8748    13% /opt
/dev/livedump      0.25      0.25    1%        4     1% /var/adm/ras/livedump
/dev/oralv         3.00      3.00    1%        4     1% /oracle
```
<h3>3.online extend file system</h3>
```
# extendlv oralv 240
lquerypv: Warning, physical volume hdisk2 is excluded since it may be
        either missing or removed.
# lsvg -l rootvg
rootvg:
LV NAME             TYPE       LPs     PPs     PVs  LV STATE      MOUNT POINT
hd5                 boot       1       1       1    closed/syncd  N/A
hd6                 paging     68      68      1    open/syncd    N/A
hd8                 jfs2log    1       1       1    open/syncd    N/A
hd4                 jfs2       2       2       1    open/syncd    /
hd2                 jfs2       18      18      1    open/syncd    /usr
hd9var              jfs2       4       4       1    open/syncd    /var
hd3                 jfs2       80      80      1    open/syncd    /tmp
hd1                 jfs2       1       1       1    open/syncd    /home
hd10opt             jfs2       4       4       1    open/syncd    /opt
hd11admin           jfs2       1       1       1    open/syncd    /admin
lg_dumplv           sysdump    16      16      1    open/syncd    N/A
livedump            jfs2       2       2       1    open/syncd    /var/adm/ras/livedump
oralv               jfs2       264     264     1    open/syncd    /oracle
# chfs -a size=+30G /oracle
Filesystem size changed to 69206016
# df -g
Filesystem    GB blocks      Free %Used    Iused %Iused Mounted on
/dev/hd4           0.25      0.07   74%    10612    40% /
/dev/hd2           2.25      0.04   99%    49260    80% /usr
/dev/hd9var        0.50      0.19   63%     8223    16% /var
/dev/hd3          10.00      9.53    5%      748     1% /tmp
/dev/hd1           0.12      0.12    1%        5     1% /home
/dev/hd11admin      0.12      0.12    1%        5     1% /admin
/proc                 -         -    -         -     -  /proc
/dev/hd10opt       0.50      0.26   48%     8748    13% /opt
/dev/livedump      0.25      0.25    1%        4     1% /var/adm/ras/livedump
/dev/oralv        33.00     32.99    1%        4     1% /oracle
```
<h3>4.create user & group</h3>
```
# mkgroup id=501 oinstall
# mkgroup id=502 dba
# mkuser id=501 pgrp='oinstall' groups=dba oracle
# id oracle
uid=501(oracle) gid=501(oinstall) groups=502(dba)
```
<h3>5.disk driver info & lun info</h3>
```
# lsattr -El hdisk0
PCM             PCM/friend/scsiscsd                        Path Control Module           False
algorithm       fail_over                                  Algorithm                     True
dist_err_pcnt   0                                          Distributed Error Percentage  True
dist_tw_width   50                                         Distributed Error Sample Time True
hcheck_interval 0                                          Health Check Interval         True
hcheck_mode     nonactive                                  Health Check Mode             True
max_coalesce    0x10000                                    Maximum Coalesce Size         True
max_transfer    0x100000                                   Maximum TRANSFER Size         True
pvid            00f8c13b65847a820000000000000000           Physical volume identifier    False
queue_depth     16                                         Queue DEPTH                   True
reserve_policy  no_reserve                                 Reserve Policy                True
size_in_mb      146800                                     Size in Megabytes             False
unique_id       2A1135000C5005EA9607B0BST9146853SS03IBMsas Unique device identifier      False
ww_id           5000c5005ea9607b                           World Wide Identifier         False
```
上述命令不能看到emc多路径LUN的大小：
```
#lsattr -El hdiskpower3
PR_key_value   none                             Reserve Key.                                   True
clr_q          no                               Clear Queue (RS/6000)                          True
location                                        Location                                       True
lun_id         0x3000000000000                  LUN ID                                         False
lun_reset_spt  yes                              FC Forced Open LUN                             True
max_coalesce   0x100000                         Maximum coalesce size                          True
max_transfer   0x100000                         Maximum transfer size                          True
pvid           00f8c13b3303bc100000000000000000 Physical volume identifier                     False
pvid_takeover  yes                              Takeover PVIDs from hdisks                     True
q_err          yes                              Use QERR bit                                   True
q_type         simple                           Queue TYPE                                     False
queue_depth    32                               Queue DEPTH                                    True
reassign_to    120                              REASSIGN time out value                        True
reserve_policy single_path                      Reserve Policy used to reserve device on open. True
rw_timeout     30                               READ/WRITE time out                            True
scsi_id        0x11600                          SCSI ID                                        False
start_timeout  60                               START unit time out                            True
ww_name        0x500601693ee02689               World Wide Name                                False
```
查看存储mapping过来的LUN大小，该LUN的VG必须要激活：
```
#lsvg -o
datavg
rootvg
#lspv
hdisk0          00f8c13c29b5f01d                    rootvg          active
hdisk2          00f8c13c55eba068                    rootvg          active
hdisk3          none                                None
hdisk4          none                                None
hdisk5          none                                None
hdisk6          none                                None
hdisk7          none                                None
hdisk8          none                                None
hdisk9          none                                None
hdisk10         none                                None
hdisk11         none                                None
hdisk12         none                                None
hdisk13         none                                None
hdisk14         none                                None
hdisk15         none                                None
hdisk16         none                                None
hdisk17         none                                None
hdisk18         none                                None
hdisk19         none                                None
hdisk20         none                                None
hdisk21         none                                None
hdisk22         none                                None
hdisk23         none                                None
hdisk24         none                                None
hdisk25         none                                None
hdisk26         none                                None
hdisk27         none                                None
hdiskpower0     00f8c13b330bcfbd                    hbvg
hdiskpower1     00f8c13b3301701a                    datavg          active
hdiskpower2     00f8c13b330502dd                    datavg          active
hdiskpower3     00f8c13b3303bc10                    datavg          active
#lspv hdiskpower1
PHYSICAL VOLUME:    hdiskpower1              VOLUME GROUP:     datavg
PV IDENTIFIER:      00f8c13b3301701a VG IDENTIFIER     00f8c13b00004c00000001446a0ea7e3
PV STATE:           active
STALE PARTITIONS:   0                        ALLOCATABLE:      yes
PP SIZE:            512 megabyte(s)          LOGICAL VOLUMES:  1
TOTAL PPs:          399 (204288 megabytes)   VG DESCRIPTORS:   1
FREE PPs:           0 (0 megabytes)          HOT SPARE:        no
USED PPs:           399 (204288 megabytes)   MAX REQUEST:      1 megabyte
FREE DISTRIBUTION:  00..00..00..00..00
USED DISTRIBUTION:  80..80..79..80..80
MIRROR POOL:        None
```
对于EMC使用powerpath mapping的LUN，可用如下命令查询：
```
#powermt display dev=all
Pseudo name=hdiskpower0
CLARiiON ID=CKM00130902402 [HACMP]
Logical device ID=600601605C80340006DBA9929B9DE311 [LUN 0]
state=alive; policy=CLAROpt; priority=0; queued-IOs=0;
Owner: default=SP A, current=SP A       Array failover mode: 4
==============================================================================
--------------- Host ---------------   - Stor -   -- I/O Path --  -- Stats ---
###  HW Path               I/O Paths    Interf.   Mode    State   Q-IOs Errors
==============================================================================
   0 fscsi0                   hdisk10   SP B0     active  alive       0      0
   1 fscsi2                   hdisk16   SP A1     active  alive       0      0
   1 fscsi2                   hdisk22   SP B1     active  alive       0      0
   0 fscsi0                   hdisk4    SP A0     active  alive       0      0

Pseudo name=hdiskpower4
CLARiiON ID=CKM00130902402 [HACMP]
Logical device ID=600601605C80340024AE82E69B9DE311 [LUN 4]
state=alive; policy=CLAROpt; priority=0; queued-IOs=0;
Owner: default=SP B, current=SP B       Array failover mode: 4
==============================================================================
--------------- Host ---------------   - Stor -   -- I/O Path --  -- Stats ---
###  HW Path               I/O Paths    Interf.   Mode    State   Q-IOs Errors
==============================================================================
   0 fscsi0                   hdisk14   SP B0     active  alive       0      0
   1 fscsi2                   hdisk20   SP A1     active  alive       0      0
   1 fscsi2                   hdisk26   SP B1     active  alive       0      0
   0 fscsi0                   hdisk8    SP A0     active  alive       0      0

Pseudo name=hdiskpower5
CLARiiON ID=CKM00130902402 [HACMP]
Logical device ID=600601605C803400705107F09B9DE311 [LUN 5]
state=alive; policy=CLAROpt; priority=0; queued-IOs=0;
Owner: default=SP A, current=SP A       Array failover mode: 4
==============================================================================
--------------- Host ---------------   - Stor -   -- I/O Path --  -- Stats ---
###  HW Path               I/O Paths    Interf.   Mode    State   Q-IOs Errors
==============================================================================
   0 fscsi0                   hdisk15   SP B0     active  alive       0      0
   1 fscsi2                   hdisk21   SP A1     active  alive       0      0
   1 fscsi2                   hdisk27   SP B1     active  alive       0      0
   0 fscsi0                   hdisk9    SP A0     active  alive       0      0

Pseudo name=hdiskpower1
CLARiiON ID=CKM00130902402 [HACMP]
Logical device ID=600601605C8034008A0968A99B9DE311 [LUN 1]
state=alive; policy=CLAROpt; priority=0; queued-IOs=0;
Owner: default=SP A, current=SP A       Array failover mode: 4
==============================================================================
--------------- Host ---------------   - Stor -   -- I/O Path --  -- Stats ---
###  HW Path               I/O Paths    Interf.   Mode    State   Q-IOs Errors
==============================================================================
   0 fscsi0                   hdisk11   SP B0     active  alive       0      0
   1 fscsi2                   hdisk17   SP A1     active  alive       0      0
   1 fscsi2                   hdisk23   SP B1     active  alive       0      0
   0 fscsi0                   hdisk5    SP A0     active  alive       0      0

Pseudo name=hdiskpower3
CLARiiON ID=CKM00130902402 [HACMP]
Logical device ID=600601605C803400963FDACC9B9DE311 [LUN 3]
state=alive; policy=CLAROpt; priority=0; queued-IOs=0;
Owner: default=SP A, current=SP A       Array failover mode: 4
==============================================================================
--------------- Host ---------------   - Stor -   -- I/O Path --  -- Stats ---
###  HW Path               I/O Paths    Interf.   Mode    State   Q-IOs Errors
==============================================================================
   0 fscsi0                   hdisk13   SP B0     active  alive       0      0
   1 fscsi2                   hdisk19   SP A1     active  alive       0      0
   1 fscsi2                   hdisk25   SP B1     active  alive       0      0
   0 fscsi0                   hdisk7    SP A0     active  alive       0      0

Pseudo name=hdiskpower2
CLARiiON ID=CKM00130902402 [HACMP]
Logical device ID=600601605C803400DA36EBC19B9DE311 [LUN 2]
state=alive; policy=CLAROpt; priority=0; queued-IOs=0;
Owner: default=SP B, current=SP B       Array failover mode: 4
==============================================================================
--------------- Host ---------------   - Stor -   -- I/O Path --  -- Stats ---
###  HW Path               I/O Paths    Interf.   Mode    State   Q-IOs Errors
==============================================================================
   0 fscsi0                   hdisk12   SP B0     active  alive       0      0
   1 fscsi2                   hdisk18   SP A1     active  alive       0      0
   1 fscsi2                   hdisk24   SP B1     active  alive       0      0
   0 fscsi0                   hdisk6    SP A0     active  alive       0      0
```
To be continued
