---
title: Hugepage setting on Linux/AIX
categories: oracle
comments: false
date: 2018-10-09 15:34:16
tags: tuning
---

Nowadays, with inexpensive hardware cost, large memory is more and more popular then before. In best practice, for the large memory, there are some recommendations.

<!--more-->
# 1. What is Hugepage
HugePages is a method to have larger pages where it is useful for working with very large memory. It is both useful in 32- and 64-bit configurations. By default, Linux kernel manage memory by dividing 4K pages, with Hugepage, the pagesize is increased to 2MB, therefore, reduce total number of pages by Linux kernel, meanwhile, it reduce the amount of memory for managing the page table. Hugepages are not swappable, which avoiding performance issue by swapping.
{% colorquote info %}
From the Oracle Database perspective, with HugePages, the Linux kernel will use less memory to create pagetables to maintain virtual to physical mappings for SGA address range, in comparison to regular size pages. This makes more memory to be available for process-private computations or PGA usage.
{% endcolorquote %}

The Hugapage concept is introduced in 2.6.23 kernel.

# 2. How to set
Be aware, Hugepage is conflict with AMM, __DO NOT use AMM while Hugepage is enabled__.
## 2.1 Linux platform
There's a script to calculate the Hugepage size in MOS: [Oracle Linux: Shell Script to Calculate Values Recommended Linux HugePages / HugeTLB Configuration (Doc ID 401749.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=324400953808653&parent=DOCUMENT&sourceId=361468.1&id=401749.1&_afrWindowMode=0&_adf.ctrl-state=v1xoqgu3h_150)

The output should be like this:
```sh
...
Recommended setting: vm.nr_hugepages = 67
```
Add above values to `/etc/sysctl.conf`, and execute `sysctl -p` to enable the configuration.

MOS: [HugePages on Oracle Linux 64-bit (Doc ID 361468.1)](https://support.oracle.com/epmos/faces/SearchDocDisplay?_adf.ctrl-state=v1xoqgu3h_209&_afrLoop=325228706963147)

* Setting memlock in `/etc/security/limits.conf`
Set the value (in KB) slightly smaller than installed RAM, example for 64G RAM:

```sh
oracle   soft   memlock    60397977
oracle   hard   memlock    60397977
```

* Verify Hugepage is set

```sh
grep -i huge /proc/meminfo
HugePages_Total:       133300      --total huge pages
HugePages_Free:        28031       --free huge pages
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

__DO NOT forget to disable AMM if enabled__. Remove `memory_target` and `memory_max_size` from spfile is recommended.

## 2.2 AIX platform
Huge page on AIX is called largepages.
MOS: [How to enable Large Page Feature on AIX-Based Systems (Doc ID 372157.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=327897114456641&id=372157.1&_adf.ctrl-state=v1xoqgu3h_666)

* Verify current large pages setting

```sh
root@db01:/# vmo -L lgpg_size
NAME                      CUR    DEF    BOOT   MIN    MAX    UNIT           TYPE
     DEPENDENCIES
--------------------------------------------------------------------------------
lgpg_size                 0      0      0      0      16M    bytes             D
     lgpg_regions
--------------------------------------------------------------------------------
root@db01:/# vmo -L lgpg_regions
NAME                      CUR    DEF    BOOT   MIN    MAX    UNIT           TYPE
     DEPENDENCIES
--------------------------------------------------------------------------------
lgpg_regions              0      0      0      0      8E-1                     D
     lgpg_size
--------------------------------------------------------------------------------
```

* Setting large pages
Give the Oracle user ID the CAP_BYPASS_RAC_VMM and CAP_PROPAGATE capabilities by following these steps:

```sh
--First check the current capabilities:
lsuser -a capabilities root
lsuser -a capabilities grid
lsuser -a capabilities oracle
--dd the CAP_BYPASS_RAC_VMM and CAP_PROPAGATE capabilities to the list of capabilities already assigned to this user ID:
chuser capabilities=CAP_NUMA_ATTACH,CAP_BYPASS_RAC_VMM,CAP_PROPAGATE oracle
chuser capabilities=CAP_NUMA_ATTACH,CAP_BYPASS_RAC_VMM,CAP_PROPAGATE root
```
Configure the number and size of large pages:
```sh
vmo -p -o lgpg_regions=3800 -o lgpg_size=16777216
```
> num_of_large_pages = INT((total_SGA_size-1)/16MB)+1
Allocate 16777216 bytes to provide large pages, with 3800 actual large pages.

Change lru_file_repage, the default is 1:
```sh
vmo -o lru_file_repage=0
```

Load the configuration into the kernel and reboot the server
```sh
bosboot -a
reboot
```

* Setting parameters in instance level

```sql
alter system set lock_sga=TRUE scope=spfile;
```
> On AIX databases, USE_LARGE_PAGES parameter has NO impact.
> These parameters are only valid for databases running on Linux, the value of this parameter even if set to FALSE will be ignored on AIX.

After starting up instance, verify large pages is using:
```sh
svmon -P SMON_PID
vmstat -P all
```


# 3. Disable transparent Hugepages in Linux
MOS: [ALERT: Disable Transparent HugePages on SLES11, RHEL6, RHEL7, OL6, OL7, and UEK2 and above (Doc ID 1557478.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=325509027121186&parent=DOCUMENT&sourceId=361468.1&id=1557478.1&_afrWindowMode=0&_adf.ctrl-state=v1xoqgu3h_433)

For Linux, Oracle strongly recommend to disable transparent Hugepages, especially in RAC, transparent Hugepages will occur node reboot.

* Verify current setting

```
cat /sys/kernel/mm/transparent_hugepage/enabled #OEL
cat //sys/kernel/mm/redhat_transparent_hugepage/enabled #RHEL
       [always] madvise never
```
If "enabled" is NOT set to "[never]", the Transparent HugePages are being used.

Or
```sh
grep AnonHugePages /proc/meminfo
```
If the AnonHugePages is not equal 0, the kernel is using Transparent HugePages.

* RHEL6
Add "transparent_hugepage=never" to the kernel boot line in the "/boot/grub/grub.conf" file.

* RHEL7
RHEL7: edit the "/boot/grub2/grub.cfg" file using the grubby command.

```bash
grubby --default-kernel
   /boot/vmlinuz-4.1.12-61.1.6.el7uek.x86_64
grubby --args="transparent_hugepage=never" --update-kernel /boot/vmlinuz-4.1.12-61.1.6.el7uek.x86_64
grubby --info /boot/vmlinuz-4.1.12-61.1.6.el7uek.x86_64
```
Server must be rebooted for this to take effect

Alternatively, add the following lines into the "/etc/rc.local" file and reboot the server.
```sh
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
   echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
```

# 4. Recommended setting for SGA > 100G in RAC
For RAC system which SGA is over 100G, Oracle provide a best practice: [Best Practices and Recommendations for RAC databases with SGA size over 100GB (Doc ID 1619155.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=326147990144319&id=1619155.1&_adf.ctrl-state=v1xoqgu3h_589)

init.ora parameters:

```sql
a. Set _lm_sync_timeout to 1200
(this recommendation is valid only for databases that are12.2 and lower)
```
   > Setting this will prevent some timeouts during reconfiguration and DRM. It's a static parameter and rolling restart is supported.

```sql
b. Set shared_pool_size to 15% or larger of the total SGA size.
```
   > For example, if SGA size is 1 TB, the shared pool size should be at least 150 GB. It's a dynamic parameter.

```sql
c. Set _gc_policy_minimum to 15000
```
   > There is no need to set `_gc_policy_minimum` if DRM is disabled by setting `_gc_policy_time = 0`. `_gc_policy_minimum` is a dynamic parameter, `_gc_policy_time` is a static parameter and rolling restart is not supported. To disable DRM, instead of `_gc_policy_time`, `_lm_drm_disable` should be used as it's dynamic.

```sql
d. Set _lm_tickets to 5000
(this recommendation is valid only for databases that are12.2 and lower)
```
   > Default is 1000.   Allocating more tickets (used for sending messages) avoids issues where we ran out of tickets during the reconfiguration. It's a static parameter and rolling restart is supported. When increasing the parameter, rolling restart is fine but a cold restart can be necessary when decreasing.

```sql
e. Set gcs_server_processes to the twice the default number of lms processes that are allocated.
(this recommendation is valid only for databases that are12.2 and lower)
```
> The default number of lms processes depends on the number of CPUs/cores that the server has, so please refer to the gcs_server_processes init.ora parameter section in the Oracle Database Reference Guide for the default number of lms processes for your server.  Please make sure that the total number of lms processes of all databases on the server is less than the total number of CPUs/cores on the server.  Please refer to the Document 558185.1 It's a static parameter and rolling restart is supported.




__EOF__
