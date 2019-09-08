---
title: Easy Connect with ORA-12504
categories: oracle
comments: false
date: 2019-08-06 08:49:22
tags: 
---
Easy connect naming method will cause below error if you didn't specify user password:
```sql
[oracle@db2srv:/home/oracle]$ sqlplus system@192.168.56.101:1522/linora

SQL*Plus: Release 12.2.0.1.0 Production on Tue Aug 6 08:48:46 2019

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

ERROR:
ORA-12504: TNS:listener was not given the SERVICE_NAME in CONNECT_DATA
```
It's because Oracle will translate service_name as user password.
The solution is using double quote and escape character for connect strings:

```sql
[oracle@db2srv:/home/oracle]$ sqlplus system@\"192.168.56.101:1522/linora\" as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Tue Aug 6 10:53:57 2019

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

Enter password:

[oracle@db2srv:/home/oracle]$ sqlplus /nolog

SQL*Plus: Release 12.2.0.1.0 Production on Tue Aug 6 10:55:53 2019

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

idle> conn system@"192.168.56.101:1522/linora"
Enter password
```



__EOF__
