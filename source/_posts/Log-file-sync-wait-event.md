---
title: Log file sync wait event
categories: oracle
comments: false
date: 2018-10-25 16:19:04
tags: tuning
---

While user commit/rollback, commit will trigger log file sync, that is lgwr writing log buffer(memory) to log file(disk), before the commit/rollback completing, user will see the `log file sync` wait event.
Quote from MOS ID:1376916.1:

{% colorquote info %}
At the time of commit, the user session will post LGWR to write the log buffer (containing the current unwritten redo, including this session's redo records) to the redo log file. Once LGWR knows that its write requests have completed, it will post the user session to notify it that this has completed. The user session waits on 'log file sync' while waiting for LGWR to post it back to confirm all redo it generated have made it safely onto disk.
{% endcolorquote %}

{% colorquote info %}
The time between the user session posting the LGWR and the LG's  posting the user after the write has completed is the wait time for 'log file sync' that the user session will show.
{% endcolorquote %}
In RAC system, `gc log flush sync` is another manifestation of `log file sync`.
<!--more-->

For diagnosing log file sync issue, we can:
1. Check the AWR report and dynamic views
2. Check the alert log, alert log shows how frequently redo log file are switching, recommended time for redo log switching is 15 mins to 30 mins
3. As of 10.2.0.4, Oracle will write the warning message("log write elapsed time xxms, xxKB") to LG's trace file when writing takes more then 500ms, if the size is very small, we can consider the I/O is poor
4. Check the hidden parameter `_use_adaptive_log_file_sync`, see its value is true or false

# 1. What cause log file sync

## 1.1 Slow Disk I/O
If `log file sync` and `log file parallel write` are approximate equal, we can consider the Disk I/O is poor.
The average wait time on `log file sync` on HDD should be less then 20ms.

Checking LGWR wait time:
```sql
select event, state, seq#, seconds_in_wait, p1,p2
from gv$session
where program like '%LGWR%';
```

In AWR report, check the statistics of `Log file parallel write per second`.

* Recommendation
   * Identify any bottle neck of I/O subsystems, including bandwidth
   * Avoid placing redo log in RAID 5/RAID 6 disk array
   * Avoid placing redo log in SSD
   * Identify any other processes are occupied too much IO operations

* Example

Below example shows slow disk IO, we can see that, the avg wait ms is over 20ms.
TOP 10 wait event:

![TOP 10 wait event](/images/log_file_sync1.jpg)

Log file parallel write:

![Log file parallel write](/images/log_file_parallel_write.jpg)

If `log file sync` are greater then `log parallel write` and `log parallel write` is within the normal time, it doesn't mean I/O subsystem works fine, peak IO may occur during the past snapshot time, if nmon or oswatcher is implemented, there should be some clues with peak IO statistics.
## 1.2 High commit activity

In the AWR or Statspack report, if the average user calls per commit/rollback calculated as "user calls/(user commits+user rollbacks)" is less than 30, then commits are happening too frequently:
Excessive commit(5.2 user calls/user commit):

![Excessive commit](/images/excessive_commits.jpg)

## 1.3 High CPUs
If the `r` column of vmstat output is higher than CPU numbers, we can consider excessive consuming. Sometimes the IO response time is slower because of high CPU usages.
## 1.4 Bugs

# 2. Relevant factors
In AWR report, some other wait events maybe company with `log file sync`. As shown previous top 10 event, `log buffer space` and `log file switch` also on the top 10 event.

## 2.1 _use_adaptive_log_file_sync
From 11.2.0.3, underscore parameter `_use_adaptive_log_file_sync` will affect log write performance because it changes the default behavior of LGWR. See [Hidden parameter '_use_adaptive_log_file_sync'](/use-adaptive-log-file-sync.html)

## 2.2 log file blocksize
Blocksize in `v$log` view shows how much size of the block size, the default size is 512 kb, on a busy oltp system, modify the size to 2M maybe is a better option.

## 2.3 log file size
First, if log file size is too small, alert.log will show lots of `log` and AWR report may show `log file switch (checkpoint incomplete)` wait event it the top 10 wait events.
Below AWR report shows excessive log switches, Oracle recommend that a log switch should occur at most once every 15 to 20 minutes.

![Excessive log switch](/images/log_switch_freq1.jpg)

Under this circumstance, resize redo log file size is a better choice(current size is the default size:50M).

Reference:
[High Waits for 'Log File Sync': Known Issue Checklist for 11.2 (Doc ID 1548261.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=446818976135672&id=1548261.1&_adf.ctrl-state=o2nxn6cvz_62)
[Troubleshooting: 'Log file sync' Waits (Doc ID 1376916.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=446901390154542&parent=DOCUMENT&sourceId=1548261.1&id=1376916.1&_afrWindowMode=0&_adf.ctrl-state=o2nxn6cvz_119)
[Script to Collect Log File Sync Diagnostic Information (lfsdiag.sql)](https://support.oracle.com/epmos/faces/DocumentDisplay?parent=DOCUMENT&sourceId=1376916.1&id=1064487.1)


__EOF__
