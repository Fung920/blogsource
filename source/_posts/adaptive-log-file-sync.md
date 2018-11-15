---
title: Adaptive log file sync
categories: oracle
comments: false
date: 2018-10-15 22:48:22
tags: tuning
---

From Oracle 11.2.0.3, hidden parameter `_use_adaptive_log_file_sync` are changed default value from FALSE to TRUE.
Quote from Oracle community:[_use_adaptive_log_file_sync](https://community.oracle.com/thread/3520420)
When `_use_adaptive_log_file_sync` is set to true, Oracle switches between two methods of communication between the LGWR and foreground processes to acknowledge that a commit has completed:
   * Post/wait - conventional method available in previous Oracle releases.
      * LGWR explicitly posts all processes waiting for the commit to complete.

   * Polling
      * Foreground processes sleep and poll to see if the commit is complete.
      * The advantage of this new method is to free LGWR from having to inform many processes waiting on commit to complete thereby freeing high CPU usage by the LGWR.
      * Initially the LGWR uses post/wait and according to an internal algorithm evaluates whether polling is better.

Under high system load polling may perform better because the post/wait implementation typically does not scale well.
If the system load is low, then post/wait performs well and provides better response times than polling.
Oracle relies on internal statistics to determine which method should be used.
Because switching between post/wait and polling incurs an overhead, safe guards are in place in order to ensure that switches do not occur too frequently.
Switch is logged in LGWR tracefile
<!--more-->
Below DB version is 12.2, should be different from 11g, the first parameter freq_threshold limits the mode switch frequency.
```sql
select a.ksppinm name, b.ksppstvl value, a.ksppdesc description
from sys.x$ksppi a, sys.x$ksppcv b
where a.indx = b.indx and a.ksppinm like '_%adaptive_log%';
NAME							VALUE	   DESCRIPTION
------------------------------------------------------- ---------- ----------------------------------------------------------------------------
_adaptive_log_file_sync_high_switch_freq_threshold	3	   Threshold for frequent log file sync mode switches (per minute)
_adaptive_log_file_sync_poll_aggressiveness		0	   Polling interval selection bias (conservative=0, aggressive=100)
_adaptive_log_file_sync_sampling_count			128	   Evaluate post/wait versus polling every N writes
_adaptive_log_file_sync_sampling_time			3	   Evaluate post/wait versus polling every N seconds
_adaptive_log_file_sync_sched_delay_window		60	   Window (in seconds) for measuring average scheduling delay
_adaptive_log_file_sync_use_polling_threshold		110	   Ratio of redo synch time to expected poll time as a percentage
_adaptive_log_file_sync_use_postwait_threshold		50	   Percentage of foreground load from when post/wait was last used
_adaptive_log_file_sync_use_postwait_threshold_aging	1001	   Permille of foreground load from when post/wait was last used
_use_adaptive_log_file_sync				FALSE	   Adaptively switch between post/wait and polling
```

From `v$sysstat` view, polling status can be checked by below sql:
```sql
sys@LINORA> select name,value from v$sysstat where name in ('redo synch poll writes','redo synch polls');
NAME							VALUE
-------------------------------------------------- ----------
redo synch poll writes					    0
redo synch polls					    0
```
From AWR, `other instance activity stats` also shows the `redo synch poll writes` and `redo synch polls` activities information.

Enabling adaptive log file sync may occur LGWR relative performance issue, from best practise, disable this feature is recommended:
```sql
alter system set "_use_adaptive_log_file_sync" = false scope=spfile sid='*';
--and recycle the instance
```

In summary, conventional way will reduce the wait time of `log file sync`, but LGWR background process will cost more overhead.
And polling way, it reduce LGWR's overhead, because the COMMIT foreground process will sleep after it inform LGWR to write the changed data into log, hence the COMMIT process will wait more time for event `log file sync`. For large OLTP system, this mechanism is not suitable.

Reference:
[Adaptive Switching Between Log Write Methods can Cause 'log file sync' Waits (Doc ID 1462942.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=439981931225234&id=1462942.1&_adf.ctrl-state=kw6ghokkn_57)
[Adaptive Log File Sync Optimization (Doc ID 1541136.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=440133138964903&id=1541136.1&_adf.ctrl-state=kw6ghokkn_114)
__EOF__
