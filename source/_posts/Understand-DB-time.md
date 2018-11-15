---
title: Understand DB time
categories: oracle
comments: false
date: 2018-10-27 23:05:01
tags: tuning
---

When talking about Oracle performance tuning, DBAs always face some indices of time dimension, such as DB time, CPU time, etc,. Time Model is a critical metric of performance tuning measure. Because, performance is always about time--response time, we tune the system and try to make it run faster. Time Model Metric can be the starting point of tuning.
<!--more-->

## About DB time
* Abstract
   As it name implies, DB time is database time total spend on handling ___user calls(that means db time not include background cpu time___. In other words, DB Time = CPU Time + IO Time + NonIdle Wait Time. DB time is not only the time spend on handling active session calls but also on waiting for some resources.
   In an AWR report, some busy systems, the DB time is greater than Elapsed time, we know Elapsed time it's the time between to snapshots. But why DB time is greater than Elapsed time? Basically this means that multiple sessions were active for the investigated time period.
   Mention about DB time, it can't ignore the Average Active Session(AAS). AAS means, between a given time intervals, how many active sessions in average. For example, if time interval is 15mins, and DB time is 30mins, the AAS = DB time / Elapsed time = 2.0, that means, there are two active sessions during the 15 mins, and each session can use a whole Elapsed time. That's why in busy system, DB time is always greater then Elapsed time. So, __Average Active Session is one of the manifestation of DB loading__. And that's why in OEM performance home page, it shows both the Average Active Session and DB time.If average active sessions passes CPU Cores limit it means that some sessions will experience wait for CPU (CPU Wait).

![OEM DB time](/images/oem_dbtime1.png)

* DB time and ASH
    `v$active_session_history`, samples all active background and foreground sessions in every second, only foreground sessions are calculated in DB time. This view's purpose is reserving 1 hours statistics data.
    `DBA_HIST_ACTIVE_SESS_HISTORY`, on the other hand, samples only 1 out of 10 seconds. This view stores duration is depending on snapshot duration setting.
    Based on above information, DB time in Seconds = :
    ```sql
    select count(*) from v$active_session_history
    where sample_time between xxx and xxx where session_type = 'FOREGROUND'
    ```
    Equals:
    ```sql
    select count(*) * 10 from dba_hist_active_sess_history
    where session_type = 'FOREGROUND'
    and sample_time between xxx and xxx;
    ```

* Other tips
    ```sql
    Inactive session = totol waits in second(SQL*Net message from client)/elasped time * 60.
    DB CPU is time running in CPU(waiting in runqueue not included, which is CPU in WAITs in OEM).
    DB CPU load = DB CPU wait time(s) /elasped time * 60
    ```

## About CPU time
In AWR report, three different names indicate CPU usage for database:

* CPU time
    Represents foreground and background processes spend on CPU, does not include time waiting on CPU.
* DB CPU
    Represents only foreground process spend on CPU.
* CPU used by this session
    Amount of CPU time (in 10s of milliseconds) used by a session from the time a user call starts until it ends. If a user call completes within 10 milliseconds, the start and end user-call time are the same for purposes of this statistics, and 0 milliseconds are added.

For calculating CPU usage in ARR, in the section `Operating System Statistics`, CPU usage%=BUSY_TIME/(BUSY_TIME+IDLE_TIME), which derived from `v$osstat`.

### High CPU usage diagnosing
* High Parse consumption
* Excessive Logical reads
    Looking at `SQL ordered by CPU time` section of AWR report to see if any excessive logical IO can be tuned or sorts can be avoided. Also check the `segments by logical reads` to see which segments are causing excessive logical IO.
* Logon storms
    Looking at `logons cumulative` statistics at AWR report to find out. Every time a new logon requires, OS need to start up a process, allocate memory for shared pool and PGA, all these activities take CPU.
* Resource manager events
    For example:  resmgr: cpu quantum
* Latch/Mutex wait events
    Latch/Mutex contention burns cpu in a high rate. In this case, `SQL ordered by CPU time` is useless, looking at ASH report to find more out.



## Relevant dynamic performance views
Based on cumulative statistics, some performance views provided different metric, such as CPU time, user call, etc,.

* v$sysmetric
    Displays the system metric values captured for the most current time interval for both the long duration (60-second) and short duration (15-second) system metrics.
    The column `group_id` represents different interval:
    > group_id=2: 60 second interval
    > group_id=3: 15 second interval
    For a single session metric, view the `v$sessmetric` instead.
* v$sysmetric_summary
    Displays the system metric average, maximum, minimum values for the last hour. Only for the long duration data.
* v$sysmetric_history
    For the last hour Oracle stores the 60 second intervals and for the 15 second intervals in this view.
* v$metricname
    Displays the mapping of the name of metrics to their metric ID.
* DBA_HIST_SYSMETRIC_HISTORY
   This view contains snapshots of V$SYSMETRIC_HISTORY. One of the source of AWR report.

Summary of metric v$ views:
   - v$sysmetric - last 15 seconds and 60 seconds
   - v$sysmetric_summary - values  last hour (last snapshot)  like avg, max, min etc
   - v$sysmetric_history - last hour for 1 minute, last 3 mintes for 15 second deltas

* v$sess_time_model
    Displays the session-wide accumulated times for various operations.
* v$sys_time_model
    Displays the system-wide accumulated times for various operations. The time reported is the total elapsed or CPU time (in microseconds). Any timed operation will buffer at most 5 seconds of time data. Specifically, this means that if a timed operation (such as SQL execution) takes a long period of time to perform, the data published to this view is at most missing 5 seconds of the time accumulated for the operation.
    Example of time model distribution:
    ```sql
    --time model tracks time in microseconds (one millionth of a second)
    select stat_name, trunc(value/1000000,2) seconds
    from v$sys_time_model
    order by 2 desc;
    ```

* v$sysstat
    Displays system statistics. To find the name of the statistic associated with each statistic number (STATISTIC#), query the `V$STATNAME` view.
* v$sesstat
    Displays user session statistics
    "CPU used by this session" from v$sesstat changes only at the end of transaction.

The corresponding value can be check here [Statistics Descriptions](https://docs.oracle.com/database/121/REFRN/GUID-2FBC1B7E-9123-41DD-8178-96176260A639.htm#REFRN-GUID-2FBC1B7E-9123-41DD-8178-96176260A639).

v$sysstat and v$sys_time_model report CPU usage of current INSTANCE only, and v$osstat report CPU usage for whole OS.




__EOF__
