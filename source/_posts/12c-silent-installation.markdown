---
layout: post
title: "12C静默安装CDB"
date: 2014-08-09 10:12:23
comments: false
categories: oracle
tags: 12c
keywords: oracle 12c installation cdb
description: how to install oracle 12c in silent mode
---
### 1. Oracle OFA结构体系
Oracle OFA全称Optimal Flexible Architecture，中文一般翻译为Oracle最优体系架构。OFA是在安装Oracle软件或者创建数据库的时候，文件夹及文件命名方式的一种标准。
<!--more-->
![Oracle OFA](/images/OFA.png)
#### 1.1 Oracle Inventory目录
此目录存储了在此服务器上安装了的Oracle软件清单，此目录必不可少，共享于多个Oracle软件版本。同时，在每个$ORACLE_HOME目录下有个inventory目录，保存了local inventory的信息。当第一次安装，Oracle会自动检测OFA结构是否带有/u[01-09]目录，如果存在，则把Inventory目录创建为类似/u01/app/oraInventory。如果Oracle安装用户环境存在ORACLE_BASE，则创建为$ORACLE_BASE同级目录下，如ORACLE_BASE为/oracle/app/oracle，则Inventory目录为/oracle/app/oraInventory。如果以上两点都不存在，那么，Inventory目录会创建在Oracle用户的$HOME目录下，如/home/oracle/oraInventory。
#### 1.2 Oracle Base目录
Oracle Base是Oracle软件安装最顶层目录，在此目录下可以安装多个版本软件，一般命名规范为/mount_point/app/software_owner。
#### 1.3 Oracle Home目录
Oracle Home目录定义指定的产品路径，如Oracle database 12c或者Oracle database 11g，每个版本的软件应该有不同的ORACLE_HOME。一般命名规范为$ORACLE_BASE/product/version/install_name。
#### 1.4 Automatic Diagnostic Repository
从11g开始，Oracle开始启用ADR，此目录包括了Oracle相关的诊断信息，包括alert log，trace文件等，一般定义为ORACLE_BASE/diag/rdbms/lower(db_unique_name)/instance_name。在10g以前，为$ORACLE_BASE/admin/db_unique_name/{a,b,c,u}ump。11g的ADR可参照前文[11g新特性—FDI简介](/oracle/11g-fault-diagnosability-infrastructure.html)。
### 2. 安装Oracle软件
oracle 12c在OS用户组上增加了几个权限，包括备份恢复组，DataGuard管理组等，具体如下所示。  
![12c OS group](/images/OSGroup.png)
在一般用途上，如果数据库管理员只有一个，为减低复杂度，建议按照10g前，只创建一个dba组即可。  
#### 2.1 创建OS用户
```
#group add -g 500
#useradd -u 500 -g oinstall -G dba ora12c
```
系统参数设定略。
#### 2.2 创建oraInst.loc文件
oraInst.loc文件内容记录了Oracle Inventory目录，Oracle软件安装和升级的OS用户组。如：
```
[root@linora u02]# cat /etc/oraInst.loc 
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
```
创建完后同时赋予权限：
```
# chown ora12c:oinstall oraInst.loc
# chmod 664 oraInst.loc
```
#### 2.3 静默安装
进入安装软件所在文件夹，查找响应文件：
```
[ora12c@linora:/worktmp]$ find . -name "*.rsp"
./database/response/dbca.rsp
./database/response/db_install.rsp
./database/response/netca.rsp
```
编辑db_install.rsp
```
[ora12c@linora:/worktmp]$ cd ./database/response/
[ora12c@linora:/worktmp/database/response]$ cat db_install.rsp | grep -v ^# | grep -v ^$ > ~/dbinst.rsp
```
对于只安装软件，修改后的配置文件如下：
```
[ora12c@linora:/home/ora12c]$ cat dbinst.rsp 
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v12.1.0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=linora
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
SELECTED_LANGUAGES=en,zh_CN
ORACLE_HOME=/u02/app/ora12c/product/12.1.0/db_1
ORACLE_BASE=/u02/app/ora12c
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=
oracle.install.db.BACKUPDBA_GROUP=dba
oracle.install.db.DGDBA_GROUP=dba
oracle.install.db.KMDBA_GROUP=dba
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
DECLINE_SECURITY_UPDATES=true 
```
开始静默安装：
```
[ora12c@linora:/worktmp/database]$ ./runInstaller -ignoreSysPrereqs -force -silent \
-responseFile /home/ora12c/dbinst.rsp
Starting Oracle Universal Installer...

Checking Temp space: must be greater than 500 MB.   Actual 4381 MB    Passed
Checking swap space: must be greater than 150 MB.   Actual 3047 MB    Passed
Preparing to launch Oracle Universal Installer from /tmp/OraInstall2014-08-09_05-42-55PM. Please wait ...
 You can find the log of this install session at:
 /u01/app/oraInventory/logs/installActions2014-08-09_05-42-55PM.log
```
在安装过程中，查看日志是否有报错。在安装完成后，按照日志提示以root用户执行脚本.
```
The installation of Oracle Database 12c was successful.
Please check '/u01/app/oraInventory/logs/silentInstall2014-08-09_05-42-55PM.log' for more details.

As a root user, execute the following script(s):
        1. /u02/app/ora12c/product/12.1.0/db_1/root.sh
```
安装过程中相关文件及路径信息。
![Useful Files for Troubleshooting Oracle Installation Issues](/images/troubleshooting.png)
### 3. 复制现有数据库软件到新机器上
这种方法能快速部署Oracle软件，从原有软件复制到新机器，或者在同一台机器复制到不同路径。由于是复制，许多目录，如dump目录，inventory目录可能需要手动创建。且因为没有全局inventory，需要重新创建。
#### 3.1 复制数据库软件
```
[ora12c@linora:/home/ora12c]$ cd $ORACLE_HOME
[ora12c@linora:/u02/app/ora12c/product/12.1.0/db_1]$ cd ../
[ora12c@linora:/u02/app/ora12c/product/12.1.0]$ tar -czvf /u02/orahome.tgz db_1/
```
通过ftp或者其他远程传输工具将文件传输至目的机器，并解压。  
也可以通过一条强大的命令直接完成以上两个需求，如：
```
$tar -cvf - db_1 | ssh ora12c "cd /u01/app/oracle/product/12c; tar -xvf -"
```
#### 3.2 重新创建全局inventory
解压后需要执行attach Oracle home：
```
[oracle@ora12c:/home/oracle]$ cd $ORACLE_HOME/oui/bin
[oracle@ora12c:/u01/app/oracle/product/12c/db_1/oui/bin]$ 
[oracle@ora12c:/u01/app/oracle/product/12c/db_1/oui/bin]$ ./runInstaller -silent \
-attachHome -invPtrLoc /etc/oraInst.loc \
ORACLE_HOME="/u01/app/oracle/product/12c/db_1" ORACLE_HOME_NAME="ONEW"
Starting Oracle Universal Installer...

Checking swap space: must be greater than 500 MB.   Actual 4095 MB    Passed
The inventory pointer is located at /etc/oraInst.loc
'AttachHome' was successful.
#查看Inventory，看看是否添加成功
[oracle@ora12c:/home/oracle]$ cat /u01/app/oraInventory/ContentsXML/inventory.xml 
<?xml version="1.0" standalone="yes" ?>
<!-- Copyright (c) 1999, 2014, Oracle and/or its affiliates.
All rights reserved. -->
<!-- Do not modify the contents of this file by hand. -->
<INVENTORY>
<VERSION_INFO>
   <SAVED_WITH>12.1.0.2.0</SAVED_WITH>
   <MINIMUM_VER>2.1.0.6.0</MINIMUM_VER>
</VERSION_INFO>
<HOME_LIST>
<HOME NAME="ONEW" LOC="/u01/app/oracle/product/12c/db_1" TYPE="O" IDX="1"/>
</HOME_LIST>
<COMPOSITEHOME_LIST>
</COMPOSITEHOME_LIST>
</INVENTORY>
```
上述命令是10g的方式创建的，11gr2以后可以省略ORACLE_HOME_NAME参数了，对于RAC而言，至少需要修复两个ORACLE_HOME，一个是RDBMS，一个是CRS(10g)或者Gird_HOME(11g)：
```
./runInstaller -silent -ignoreSysPrereqs -attachHome ORACLE_HOME="<Ora_Crs_Home
Path>" ORACLE_HOME_NAME="<Name of oracleCRSHome>" LOCAL_NODE='node1'
CLUSTER_NODES=node1,node2 CRS=true
./runInstaller -silent -ignoreSysPrereqs -attachHome ORACLE_HOME="<Oracle_Home
Path>" ORACLE_HOME_NAME="<Name of oracleHome>" LOCAL_NODE='node1'
CLUSTER_NODES=node1,node2
```
### 4. 创建数据库
数据库的创建，可以通过DBCA模版或者在sqlplus手工创建。  
首先添加/etc/oratab
```
# Entries are of the form:
#   $ORACLE_SID:$ORACLE_HOME:<N|Y>:
linora:/u01/app/oracle/product/11gr2:N
ora12c:/u02/app/ora12c/product/12.1.0/db_1:N
```
#### 4.1 手工创建实例[推荐]
首先创建pfile，然后创建spfile，从spfile启动.pfile内容如下：
```
db_name=ora12c
db_block_size=8192
memory_target=800M
memory_max_target=800M
processes=300
control_files=(/u02/oradata/ora12c/control01.ctl,/u02/oradata/ora12c/control02.ctl)
job_queue_processes=10
open_cursors=500
undo_management=AUTO
undo_tablespace=UNDOTBS1
remote_login_passwordfile=EXCLUSIVE
enable_pluggable_database=true
compatible ='12.0.0'
diagnostic_dest=/u02/app/ora12c
```
在nomount状态下执行创建CDB数据库命令：
```
[ora12c@linora:/home/ora12c]$ cat dbcreate.sql 
CREATE DATABASE ora12c
MAXLOGFILES 16
MAXLOGMEMBERS 4
MAXDATAFILES 1024
MAXINSTANCES 1
MAXLOGHISTORY 680
CHARACTER SET AL32UTF8
DATAFILE '/u02/oradata/ora12c/system01.dbf' SIZE 100M autoextend on next 64M maxsize unlimited
EXTENT MANAGEMENT LOCAL
UNDO TABLESPACE undotbs1 DATAFILE '/u02/oradata/ora12c/undotbs01.dbf' SIZE 100M autoextend on next 64M maxsize unlimited
SYSAUX DATAFILE '/u02/oradata/ora12c/sysaux01.dbf' SIZE 100M autoextend on next 64M maxsize unlimited
DEFAULT TEMPORARY TABLESPACE TEMP TEMPFILE '/u02/oradata/ora12c/temp01.dbf' SIZE 10M autoextend on next 64M maxsize unlimited
DEFAULT TABLESPACE USERS DATAFILE '/u02/oradata/ora12c/users01.dbf' SIZE 20M autoextend on next 64M maxsize unlimited
LOGFILE GROUP 1 ('/u02/oradata/ora12c/redo01a.rdo') SIZE 50M,
GROUP 2 ('/u02/oradata/ora12c/redo02a.rdo') SIZE 50M,
GROUP 3 ('/u02/oradata/ora12c/redo03a.rdo') SIZE 50M
USER sys IDENTIFIED BY oracle
USER system IDENTIFIED BY oracle
  ENABLE PLUGGABLE DATABASE
    SEED
    FILE_NAME_CONVERT = ('/u02/oradata/ora12c/', 
                         '/u02/oradata/ora12c/pdbseed/')
    SYSTEM DATAFILES SIZE 125M AUTOEXTEND ON NEXT 10M MAXSIZE UNLIMITED
    SYSAUX DATAFILES SIZE 100M
  USER_DATA TABLESPACE usertbs
    DATAFILE '/u02/oradata/ora12c/pdbseed/usertbs01.dbf'
    SIZE 200M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
```
对以上语句部分解析：
<li>ENABLE PLUGGABLE DATABASE</li>
表示使用多租户环境，允许建立可插拔数据库，Oracle建立CDB$ROOT后，会自动建立一个只读的PDB seed(PDB$SEED)。如果创建non-CDB，则不需要添加Enable PDB子句。
<li>SEED</li>
开始指定PDB$SEED相关设定，Oracle使用OMF或者使用'FILE_NAME_CONVERT'决定system、sysaux及default temp tablespace文件名称，在将来新建每一个PDB，可以透过复制PDB$SEED快速建立PDB。
<li>FILE_NAME_CONVERT</li>
指定SEED数据文件与root数据文件转换，如上例中，Oracle会将CDB$ROOT system表空间数据文件'/u02/oradata/ora12c/system01.dbf'复制为PDB$SEED system表空间数据文件'/u02/oradata/ora12c/pdbseed/system01.dbf'，如果使用OMF，将会忽略这一语句。  
上述的例子中，Oracle会创建如下的表空间及数据文件：
```
#root(CDB$ROOT)
SYSTEM
SYSAUX
USERS (Default permanent tablespace)
UNDOTBS1 (undo tablespace ，整個CDB共用)
TEMP (Default temporary tablespace)
#seed(PDB$SEED)
SYSTEM
SYSAUX
DEFTBS(Default permanent tablespace)
USERTBS
TEMP(Default temporary tablespace)
```
创建Database后可查看如下信息
```
#CDB和PDB信息
SYS@ora12c> col name for a10
SYS@ora12c> col open_mode for a10
SYS@ora12c> select con_id,dbid,name,open_mode,total_size from v$containers;
    CON_ID       DBID NAME       OPEN_MODE  TOTAL_SIZE
---------- ---------- ---------- ---------- ----------
         1  233213762 CDB$ROOT   READ WRITE          0
         2  104991688 PDB$SEED   READ ONLY   466616320
#CDB和PDB表空间信息
SYS@ora12c> select con_id,ts#,name from v$tablespace order by 1,2;
    CON_ID        TS# NAME
---------- ---------- ----------
         1          0 SYSTEM
         1          1 SYSAUX
         1          2 UNDOTBS1
         1          3 TEMP
         1          4 USERS
         2          0 SYSTEM
         2          1 SYSAUX
         2          2 TEMP
         2          3 USERS
         2          4 USERTBS
#数据库文件信息
SYS@ora12c> set linesize 110
SYS@ora12c> col name for a85
SYS@ora12c> col con_id for 99999
SYS@ora12c> select con_id,name from v$datafile order by 1;
CON_ID NAME
------ -----------------------------------------------------
     1 /u02/oradata/ora12c/sysaux01.dbf
     1 /u02/oradata/ora12c/users01.dbf
     1 /u02/oradata/ora12c/undotbs01.dbf
     1 /u02/oradata/ora12c/system01.dbf
     2 /u02/oradata/ora12c/pdbseed/users01.dbf
     2 /u02/oradata/ora12c/pdbseed/usertbs01.dbf
     2 /u02/oradata/ora12c/pdbseed/system01.dbf
     2 /u02/oradata/ora12c/pdbseed/sysaux01.dbf
SYS@ora12c> select con_id,name from v$tempfile order by 1;
CON_ID NAME
------ -----------------------------------------------------
     1 /u02/oradata/ora12c/temp01.dbf
     2 /u02/oradata/ora12c/pdbseed/temp01.dbf
SYS@ora12c> col member for a80
SYS@ora12c> select con_id,member from v$logfile;
CON_ID MEMBER
------ ------------------------------------------------------
     0 /u02/oradata/ora12c/redo01a.rdo
     0 /u02/oradata/ora12c/redo02a.rdo
     0 /u02/oradata/ora12c/redo03a.rdo
SYS@ora12c> select con_id,name from v$controlfile;
CON_ID NAME
------ ------------------------------------------------------
     0 /u02/oradata/ora12c/control01.ctl
     0 /u02/oradata/ora12c/control02.ctl
```
实例创建完后，创建数据字典：
```
SQL> @?/rdbms/admin/catcdb.sql
```
按照官方文档执行完之后，发现数据字典依旧没有生成。使用以下方式创建(以oracle用户执行)：
```
[ora12c@linora:/home/ora12c]$ cd $ORACLE_HOME/rdbms/admin
[ora12c@linora:/home/ora12c]$ $ORACLE_HOME/perl/bin/perl catcon.pl -u sys/oracle -s -e -d \
$ORACLE_HOME/rdbms/admin -b catalog1 catalog.sql > ~/.catcon-catalog.log
[ora12c@linora:/home/ora12c]$ $ORACLE_HOME/perl/bin/perl catcon.pl -u sys/oracle -s -e -d \
$ORACLE_HOME/rdbms/admin -b catproc1 catproc.sql > ~/.catcon-catproc.log
[ora12c@linora:/home/ora12c]$ $ORACLE_HOME/perl/bin/perl catcon.pl -u system/oracle -s -e -d \
$ORACLE_HOME/sqlplus/admin -b pupbld1 pupbld.sql > ~/.catcon-pupbld.log
```
#### 4.2 dbca静默建库
修改默认dbca.rsp文件
```
[ora12c@linora:/home/ora12c]$ cat dbca.rsp 
[GENERAL]
RESPONSEFILE_VERSION = "12.1.0"
OPERATION_TYPE = "createDatabase"
[CREATEDATABASE]
GDBNAME = "ora12c"
SID = "ora12c"
CREATEASCONTAINERDATABASE =true
TEMPLATENAME = "General_Purpose.dbc"
SYSPASSWORD = "oracle"
SYSTEMPASSWORD = "oracle"
DATAFILEDESTINATION =/u02/oradata/
CHARACTERSET = "AL32UTF8"
NATIONALCHARACTERSET= "AL16UTF16"
```
执行创建时候报错，因为本例中的/etc/oratab已经存在SID为ora12c的条目，删除后重新执行命令:
```
[ora12c@linora:/home/ora12c]$ dbca -silent -createDatabase -responseFile ./dbca.rsp 
Look at the log file "/u02/app/ora12c/cfgtoollogs/dbca/ora12c0.log" for further details.
[ora12c@linora:/home/ora12c]$ cat /u02/app/ora12c/cfgtoollogs/dbca/ora12c0.log

The Oracle system identifier(SID) "ora12c" already exists. Specify another SID.

/u02/ has enough space. Required space is 6900 MB , available space is 13407 MB.
File Validations Successful.
[ora12c@linora:/home/ora12c]$ vi /etc/oratab 
linora:/u01/app/oracle/product/11gr2:N
#ora12c:/u02/app/ora12c/product/12.1.0/db_1:N
[ora12c@linora:/home/ora12c]$ dbca -silent -createDatabase -responseFile ./dbca.rsp 
Copying database files
1% complete
3% complete
11% complete
18% complete
26% complete
37% complete
Creating and starting Oracle instance
40% complete
45% complete
46% complete
47% complete
52% complete
57% complete
58% complete
59% complete
62% complete
Completing Database Creation
66% complete
70% complete
74% complete
85% complete
96% complete
100% complete
Look at the log file "/u02/app/ora12c/cfgtoollogs/dbca/ora12c/ora12c.log" for further details.
```
#### 4.3 创建密码文件
使用dbca创建会自动生成密码文件
```
orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID \
password=oracle entries=5 force=y ignorecase=y
```
#### 4.4 配置监听
```
[ora12c@linora:/home/ora12c]$ cat $ORACLE_HOME/network/admin/sqlnet.ora
NAMES.DIRECTORY_PATH= (TNSNAMES)

ADR_BASE = /u02/app/ora12c

[ora12c@linora:/home/ora12c]$ cat $ORACLE_HOME/network/admin/listener.ora
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = linora)(PORT = 1522))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1522))
    )
  )

ADR_BASE_LISTENER = /u02/app/ora12c

SID_LIST_LISTENER =
   (SID_DESC =
     (GLOBAL_DBNAME = ora12c)
     (ORACLE_HOME = /u02/app/ora12c/product/12.1.0/db_1)
     (SID_NAME = ora12c)
     )
```
#### 4.5 配置EM Express
```
[ora12c@linora:/home/ora12c]$ cat $ORACLE_HOME/network/admin/tnsnames.ora
ora12c =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.188)(PORT = 1522))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ora12c)
    )
  )
SYS@ora12c> show parameter service
NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
service_names                        string                 ora12c
SYS@ora12c> show parameter local_listener
NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
local_listener                       string
SYS@ora12c> alter system set local_listener=ora12c scope=both;--此处选择的是tns连接名，而不是Service name
System altered.
SYS@ora12c> alter system set dispatchers="(PROTOCOL=TCP)" scope=both;
System altered.
SYS@ora12c> exec DBMS_XDB_CONFIG.SETHTTPSPORT(5500);
PL/SQL procedure successfully completed.
```
#### 4.6 修改归档模式
```
SYS@ora12c> alter system set LOG_ARCHIVE_DEST_1 = 'LOCATION=/u02/arch' scope=both;
SYS@ora12c> shutdown immediate
SYS@ora12c> startup mount
SYS@ora12c> alter database archivelog;
SYS@ora12c> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /u02/arch
Oldest online log sequence     28
Next log sequence to archive   30
Current log sequence           30
```
最后根据需求，调整对应参数，PDB的管理在下一篇文章会提及。  
Reference:  
[Oracle® Database Administrator's Guide 12c Release 1 (12.1)](http://docs.oracle.com/database/121/ADMIN/cdb_create.htm#ADMIN13529)  
Pro Oracle Database 12c Administration, 2nd Edition
