---
title: A brief introduction of MGMTDB
categories: oracle
comments: false
date: 2019-07-31 20:23:40
tags: concept
---
From Oracle 12.1.0.2, MGMTDB is a mandatory database installed by Oracle itself.
The MGMTDB is a built-in database to store Grid Infrastructure Management Repository (GIMR). This repository enables such features as Cluster Health Monitor(aka CHM/OS, ora.crf), Oracle Database QoS Management, and Rapid Home Provisioning, and provides a historical metric repository that simplifies viewing of past performance and diagnosis of issues.
<!--more-->
Management Repository is a single instance database that's managed by Oracle Clusterware in 12c. As it's a single instance database, it will be up and running on one node in the cluster; as it's managed by GI, in case the hosting node is down, the database will be automatically failed over to other node. In 12.2, MGMTLSNR also listens on public network, therefore ora.MGMTLSNR has dependency on VIP resource type, both MGMTLSNR and MGMTDB will failover when public network is down.


In 12.1, by default, Management database uses the same shared storage as OCR/Voting File; in 12.2, fresh installation allows separate diskgroup for it.

* What's the implications of not configuring Management Database during installation/upgrade?
    In 12.1.0.1, GIMR is optional, if Management Database is not selected to be configured during installation/upgrade, all features (Cluster Health Monitor (CHM/OS) etc) that depend on it will be disabled.

    This changed in 12.1.0.2, it's mandatory to have GIMR and it's not supported to be turned off with the exception of Exadata.

    Starting with Oracle Grid Infrastructure 19c, the Grid Infrastructure Management Repository (GIMR) is optional for new installations of Oracle Standalone Cluster. Oracle Domain Services Clusters still require
the installation of a GIMR as a service component.

* How to start Management Database if it's down?
    Management Database is managed by GI and should be up and running all the time automatically. In case it's down for some reason, the following srvctl command can be used to start it:
    ```
    export ORACLE_SID=-MGMTDB
    srvctl start mgmtdb
    srvctl start mgmtlsnr
    ```

* How to "cd" to Management Database subdirectory to review trace file etc?

   ```
   cd -MGMTDB
   -bash: cd: -M: invalid option
   cd: usage: cd [-L|-P] [dir]
   
   cd ./-MGMTDB        ==>> this will work as "./" is specified
   
   more -MGMTDB_m000_9912.trc
   more: unknown option "-M"
   usage: more [-dflpcsu] [+linenum | +/pattern] name1 name2 ...
   
   more ./-MGMTDB_m000_9912.trc
   ```

* How much (shared) disk space should be allocated for the Management Database?
    Minimum:  At least 5.2 GB for the OCR volume that contains the Grid Infrastructure Management Repository (4.5 GB + 300 MB voting files + 400 MB OCR), plus 500 MB for each node for clusters greater than four nodes. For example, a six-node cluster allocation should be 6.2 GB.

* What is the default MGMT listener port in 12.2?
    MGMT listener default is 1525 in 12.2 version.



Repost from:
[FAQ: 12c Grid Infrastructure Management Repository (GIMR) (Doc ID 1568402.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=164955974638136&parent=DOCUMENT&sourceId=2065175.1&id=1568402.1&_afrWindowMode=0&_adf.ctrl-state=9e9ta6if6_58)


__EOF__

