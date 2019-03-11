---
title: 'Error for Oracle 12c in AIX rtld: 0712-001'
categories: oracle
comments: false
date: 2019-03-11 10:00:50
tags:
---

During HA testing, I shutdown one of node of RAC in AIX, after rebooting, the node couldn't join the cluster. I tried to start it manually, but failed with below issue:

```sh
exec(): 0509-036 Cannot load program crsctl because of the following errors:

rtld: 0712-001 Symbol CreateIoCompletionPort was referenced from module /opt/oracle/product/12.1.0/lib/libttsh12.so(), but a runtime definition of the symbol was not found.
```

This is due to incorrect setting of AIX IOCP.
* Check if the IOCP installed:

```sh
[root@finmdb01a /]#  lslpp -l bos.iocp.rte
  Fileset                      Level  State      Description
  ----------------------------------------------------------------------------
Path: /usr/lib/objrepos
  bos.iocp.rte               7.2.3.0  COMMITTED  I/O Completion Ports API

Path: /etc/objrepos
bos.iocp.rte               7.2.3.0  COMMITTED  I/O Completion Ports API
```

* Change IOCP setting from `smitty`

```sh
# smitty iocp
Select Change / Show Characteristics of I/O Completion Ports.
Change configured state at system restart from Defined to Available
```

* Check the status of IOCP after rebooting

Should be Available status.
```sh
[root@finmdb01a /]#  lsdev -Cc iocp
iocp0 Available  I/O Completion Ports
```

This is a prerequisite to change IOCP parameter before installing database 12c in IBM AIX environment.






__EOF__
