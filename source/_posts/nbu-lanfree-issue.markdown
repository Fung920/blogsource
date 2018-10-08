---
layout: post
title: "nbu lanfree issue"
date: 2014-04-28 09:38:08
comments: false
categories: nbu
tags: nbu
keywords: nbu,netbackup
description: lanfree in netbackup
---
客户一个新系统，两台AIX组成HACMP集群，存储挂了一个EMC VNX5500，VNX5300为备份专用，目前挂载在nbu master server上。
<!--more-->
要求通过RMAN同时往磁带和磁盘里面写。由于lanfree架构，需要在备份客户端配置san media server，即这两台小机。在NBU备份策略中，如果选取多份备份，media server需为同一台。因此，这种方案无法实现，否则就只能通过lan备份。
最后决定同一个实例配置两个policy，一个为磁带lanfree，一个为nbu master server本地磁盘。然后神奇的事情出现了，当第一个policy配置为lanfree时候，其他policy都申请lanfree资源，policy storage明明选择了disk资源，都会备份到磁带。而且其他policy调用子进程时候，显示的是第一个policy的名称。
以下截图情况说明：djgisimg为第一个policy，配置的是oracle_unit的磁盘存储。djoracle配置的是lanfree资源，备份到磁带的。
oracle_unit为磁盘资源：
![nbu1](/images/nbu1.png)
父进程名称为djoracle，子进程却变成了第一个policy名称：djgisimg。
![nbu2](/images/nbu2.png)
切父进程调用的是lanfree资源，但子进程却使用了oracle_unit资源：
![nbu2](/images/nbu3.png)
![nbu2](/images/nbu4.png)
通过官方文档，找到一个解决办法，就是在rman中，加入运行时候环境，指定NBU master server及policy。如：
```
run {
allocate channel t1 type 'SBT_TAPE';
allocate channel t2 type 'SBT_TAPE';
send 'NB_ORA_POLICY=your_policy, NB_ORA_SERV=your_server';
backup
(database format 'bk_%U_%t');
}
```
或者：
```
run {
allocate channel t1 type 'SBT_TAPE'
parms="ENV=(NB_ORA_POLICY=your_pol,
NB_ORA_SERV=your_server)";
allocate channel t2 type 'SBT_TAPE'
parms="ENV=(NB_ORA_POLICY=your_pol,
NB_ORA_SERV=your_server)";
backup
(database format 'bk_%s_%p_%t');
}
```
<code>send</code> 命令或 <code>parms</code> 操作数用于指定在备份或还原过程中要使用的 Netbackup for Oracle 环境变量。
Netbackup for Oracle的环境变量有如下参数（部分）：
<code>NB_ORA_SERV</code>:指定netbackup master server名称。
<code>NB_ORA_CLIENT</code>:指定Oracle客户端名称。
<code>NB_ORA_POLICY</code>:指定Oracle备份时所用到的备份策略。
<code>NB_ORA_SCHED</code>:指定Oracle要用到的“应用程序备份”日程表的名称。
而导致上述情况出现，是跟NBU中oracle备份策略自动产生的default-application-backup有关。按照官网的说法，指定<code>NB_ORA_SCHED</code>不走default-application-backup也可以达到目的，这种方法尚未测试。
在网上找了许多文档，也没看到官方对这个default-application-backup有一个明确的说法。个人理解，它是允许用户在客户端通过其他工具对master server发起作业，如Oracle中的RMAN工具。
此外，在NBU7.5自带的Windows脚本中，好像有点问题，执行的时候一闪而过，并没有执行备份。后面找了一个5.x的脚本，顺利通过。
附上修改后的NBU for Linux/UNIX脚本及for Windows脚本：
Linux/UNIX:
```
#!/bin/sh
# $Header: hot_database_backup.sh,v 1.3 2010/08/04 17:56:02 $
#
#bcpyrght
#***************************************************************************
#* $VRTScprght: Copyright 1993 - 2012 Symantec Corporation, All Rights Reserved $ *
#***************************************************************************
#ecpyrght
#
# ---------------------------------------------------------------------------
#                       hot_database_backup.sh
# ---------------------------------------------------------------------------
#  This script uses Recovery Manager to take a hot (inconsistent) database
#  backup. A hot backup is inconsistent because portions of the database are
#  being modified and written to the disk while the backup is progressing.
#  You must run your database in ARCHIVELOG mode to make hot backups. It is
#  assumed that this script will be executed by user root. In order for RMAN
#  to work properly we switch user (su -) to the oracle dba account before
#  execution. If this script runs under a user account that has Oracle dba
#  privilege, it will be executed using this user's account.
# ---------------------------------------------------------------------------

# ---------------------------------------------------------------------------
# Determine the user which is executing this script.
# ---------------------------------------------------------------------------

CUSER=`id |cut -d"(" -f2 | cut -d ")" -f1`

# ---------------------------------------------------------------------------
# Put output in <this file name>.out. Change as desired.
# Note: output directory requires write permission.
# ---------------------------------------------------------------------------

RMAN_LOG_FILE=${0}_`date +%y%m%d`.out

# ---------------------------------------------------------------------------
# You may want to delete the output file so that backup information does
# not accumulate.  If not, delete the following lines.
# ---------------------------------------------------------------------------

if [ -f "$RMAN_LOG_FILE" ]
then
        rm -f "$RMAN_LOG_FILE"
fi

# -----------------------------------------------------------------
# Initialize the log file.
# -----------------------------------------------------------------

echo >> $RMAN_LOG_FILE
chmod 666 $RMAN_LOG_FILE

# ---------------------------------------------------------------------------
# Log the start of this script.
# ---------------------------------------------------------------------------

echo Script $0 >> $RMAN_LOG_FILE
echo ==== started on `date` ==== >> $RMAN_LOG_FILE
echo >> $RMAN_LOG_FILE

# ---------------------------------------------------------------------------
# Replace /db/oracle/product/ora102, below, with the Oracle home path.
# ---------------------------------------------------------------------------

ORACLE_HOME=/u01/product/11.2.0/db_1
export ORACLE_HOME

# ---------------------------------------------------------------------------
# Replace ora102, below, with the Oracle SID of the target database.
# ---------------------------------------------------------------------------

ORACLE_SID=your_sid
export ORACLE_SID

# ---------------------------------------------------------------------------
# Replace ora102, below, with the Oracle DBA user id (account).
# ---------------------------------------------------------------------------

ORACLE_USER=oracle

# ---------------------------------------------------------------------------
# Set the target connect string.
# Replace "sys/manager", below, with the target connect string.
# ---------------------------------------------------------------------------

TARGET_CONNECT_STR=/

# ---------------------------------------------------------------------------
# Set the Oracle Recovery Manager name.
# ---------------------------------------------------------------------------

RMAN=$ORACLE_HOME/bin/rman

# ---------------------------------------------------------------------------
# Print out the value of the variables set by this script.
# ---------------------------------------------------------------------------

echo >> $RMAN_LOG_FILE
echo   "RMAN: $RMAN" >> $RMAN_LOG_FILE
echo   "ORACLE_SID: $ORACLE_SID" >> $RMAN_LOG_FILE
echo   "ORACLE_USER: $ORACLE_USER" >> $RMAN_LOG_FILE
echo   "ORACLE_HOME: $ORACLE_HOME" >> $RMAN_LOG_FILE

# ---------------------------------------------------------------------------
# Print out the value of the variables set by bphdb.
# ---------------------------------------------------------------------------

echo  >> $RMAN_LOG_FILE
echo   "NB_ORA_FULL: $NB_ORA_FULL" >> $RMAN_LOG_FILE
echo   "NB_ORA_INCR: $NB_ORA_INCR" >> $RMAN_LOG_FILE
echo   "NB_ORA_CINC: $NB_ORA_CINC" >> $RMAN_LOG_FILE
echo   "NB_ORA_SERV: $NB_ORA_SERV" >> $RMAN_LOG_FILE
echo   "NB_ORA_POLICY: $NB_ORA_POLICY" >> $RMAN_LOG_FILE

# ---------------------------------------------------------------------------
# NOTE: This script assumes that the database is properly opened. If desired,
# this would be the place to verify that.
# ---------------------------------------------------------------------------

echo >> $RMAN_LOG_FILE
# ---------------------------------------------------------------------------
# If this script is executed from a NetBackup schedule, NetBackup
# sets an NB_ORA environment variable based on the schedule type.
# The NB_ORA variable is then used to dynamically set BACKUP_TYPE
# For example, when:
#     schedule type is                BACKUP_TYPE is
#     ----------------                --------------
# Automatic Full                     INCREMENTAL LEVEL=0
# Automatic Differential Incremental INCREMENTAL LEVEL=1
# Automatic Cumulative Incremental   INCREMENTAL LEVEL=1 CUMULATIVE
#
# For user initiated backups, BACKUP_TYPE defaults to incremental
# level 0 (full).  To change the default for a user initiated
# backup to incremental or incremental cumulative, uncomment
# one of the following two lines.
# BACKUP_TYPE="INCREMENTAL LEVEL=1"
# BACKUP_TYPE="INCREMENTAL LEVEL=1 CUMULATIVE"
#
# Note that we use incremental level 0 to specify full backups.
# That is because, although they are identical in content, only
# the incremental level 0 backup can have incremental backups of
# level > 0 applied to it.
# ---------------------------------------------------------------------------

if [ "$NB_ORA_FULL" = "1" ]
then
        echo "Full backup requested" >> $RMAN_LOG_FILE
        BACKUP_TYPE="INCREMENTAL LEVEL=0"

elif [ "$NB_ORA_INCR" = "1" ]
then
        echo "Differential incremental backup requested" >> $RMAN_LOG_FILE
        BACKUP_TYPE="INCREMENTAL LEVEL=1"

elif [ "$NB_ORA_CINC" = "1" ]
then
        echo "Cumulative incremental backup requested" >> $RMAN_LOG_FILE
        BACKUP_TYPE="INCREMENTAL LEVEL=1 CUMULATIVE"

elif [ "$BACKUP_TYPE" = "" ]
then
        echo "Default - Full backup requested" >> $RMAN_LOG_FILE
        BACKUP_TYPE="INCREMENTAL LEVEL=0"
fi


# ---------------------------------------------------------------------------
# Call Recovery Manager to initiate the backup. This example does not use a
# Recovery Catalog. If you choose to use one, replace the option 'nocatalog'
# from the rman command line below with the
# 'catalog <userid>/<passwd>@<net service name>' statement.
#
# Note: Any environment variables needed at run time by RMAN
#       must be set and exported within the switch user (su) command.
# ---------------------------------------------------------------------------
#  Backs up the whole database.  This backup is part of the incremental
#  strategy (this means it can have incremental backups of levels > 0
#  applied to it).
#
#  We do not need to explicitly request the control file to be included
#  in this backup, as it is automatically included each time file 1 of
#  the system tablespace is backed up (the inference: as it is a whole
#  database backup, file 1 of the system tablespace will be backed up,
#  hence the controlfile will also be included automatically).
#
#  Typically, a level 0 backup would be done at least once a week.
#
#  The scenario assumes:
#     o you are backing your database up to two tape drives
#     o you want each backup set to include a maximum of 5 files
#     o you wish to include offline datafiles, and read-only tablespaces,
#       in the backup
#     o you want the backup to continue if any files are inaccessible.
#     o you are not using a Recovery Catalog
#     o you are explicitly backing up the control file.  Since you are
#       specifying nocatalog, the controlfile backup that occurs
#       automatically as the result of backing up the system file is
#       not sufficient; it will not contain records for the backup that
#       is currently in progress.
#     o you want to archive the current log, back up all the
#       archive logs using two channels, putting a maximum of 20 logs
#       in a backup set, and deleting them once the backup is complete.
#
#  Note that the format string is constructed to guarantee uniqueness and
#  to enhance NetBackup for Oracle backup and restore performance.
#
#
#  NOTE WHEN USING NET SERVICE NAME: When connecting to a database
#  using a net service name, you must use a send command or a parms operand to
#  specify environment variables.  In other words, when accessing a database
#  through a listener, the environment variables set at the system level are not
#  visible when RMAN is running.  For more information on the environment
#  variables, please refer to the NetBackup for Oracle Admin. Guide.
#
# ---------------------------------------------------------------------------

CMD_STR="
ORACLE_HOME=$ORACLE_HOME
export ORACLE_HOME
ORACLE_SID=$ORACLE_SID
export ORACLE_SID
$RMAN target $TARGET_CONNECT_STR nocatalog msglog $RMAN_LOG_FILE append << EOF
RUN {
ALLOCATE CHANNEL ch00 TYPE 'SBT_TAPE';
ALLOCATE CHANNEL ch01 TYPE 'SBT_TAPE';
send 'NB_ORA_POLICY=your_policy, NB_ORA_SERV=your_server’;
BACKUP
    $BACKUP_TYPE
    SKIP INACCESSIBLE
    TAG hot_db_bk_level0
    FILESPERSET 5
    # recommended format
    FORMAT 'bk_%s_%p_%t'
    DATABASE;
    sql 'alter system archive log current';
RELEASE CHANNEL ch00;
RELEASE CHANNEL ch01;
# backup all archive logs
ALLOCATE CHANNEL ch00 TYPE 'SBT_TAPE';
ALLOCATE CHANNEL ch01 TYPE 'SBT_TAPE';
BACKUP
   filesperset 20
   FORMAT 'al_%s_%p_%t'
   ARCHIVELOG ALL DELETE INPUT;
RELEASE CHANNEL ch00;
RELEASE CHANNEL ch01;
#
# Note: During the process of backing up the database, RMAN also backs up the
# control file.  This version of the control file does not contain the
# information about the current backup because "nocatalog" has been specified.
# To include the information about the current backup, the control file should
# be backed up as the last step of the RMAN section.  This step would not be
# necessary if we were using a recovery catalog or auto control file backups.
#
ALLOCATE CHANNEL ch00 TYPE 'SBT_TAPE';
BACKUP
    # recommended format
    FORMAT 'cntrl_%s_%p_%t'
    CURRENT CONTROLFILE;
RELEASE CHANNEL ch00;
}
EOF
"
# Initiate the command string

if [ "$CUSER" = "root" ]
then
    su - $ORACLE_USER -c "$CMD_STR" >> $RMAN_LOG_FILE
    RSTAT=$?
else
    /usr/bin/sh -c "$CMD_STR" >> $RMAN_LOG_FILE
    RSTAT=$?
fi

# ---------------------------------------------------------------------------
# Log the completion of this script.
# ---------------------------------------------------------------------------

if [ "$RSTAT" = "0" ]
then
    LOGMSG="ended successfully"
else
    LOGMSG="ended in error"
fi

echo >> $RMAN_LOG_FILE
echo Script $0 >> $RMAN_LOG_FILE
echo ==== $LOGMSG on `date` ==== >> $RMAN_LOG_FILE
echo >> $RMAN_LOG_FILE

exit $RSTAT
```
Windows NBU5.x:
```
@REM $Header: hot_database_backup.cmd,v 1.3 from QaSanil 2005/11/28 19:01:53 $

@REM bcpyrght
@REM ***************************************************************************
@REM * $VRTScprght: Copyright 1993 - 2009 Symantec Corporation, All Rights Reserved $ *
@REM ***************************************************************************
@REM ecpyrght
@REM
@REM ---------------------------------------------------------------------------
@REM                          hot_database_backup.cmd
@REM ---------------------------------------------------------------------------
@REM This script uses Recovery Manager to take a hot (inconsistent) database
@REM backup. A hot backup is inconsistent because portions of the database are
@REM being modified and written to the disk while the backup is progressing.
@REM You must run your database in ARCHIVELOG mode to make hot backups.
@REM
@REM NOTE information for running proxy backups has been included.  These
@REM information sections begin with a comment line of PROXY
@REM ---------------------------------------------------------------------------

@setlocal ENABLEEXTENSIONS

@REM ---------------------------------------------------------------------------
@REM No need to echo the commands.
@REM ---------------------------------------------------------------------------

@echo off

@REM ---------------------------------------------------------------------------
@REM Put output in the same filename, different extension.
@REM ---------------------------------------------------------------------------

@set RMAN_LOG_FILE="%~dpn0.out"

@REM ---------------------------------------------------------------------------
@REM You may want to delete the output file so that backup information does
@REM not accumulate.  If not, delete the following command.
@REM ---------------------------------------------------------------------------

@if exist %RMAN_LOG_FILE% del %RMAN_LOG_FILE%

@REM ---------------------------------------------------------------------------
@REM Replace H:\oracle\ora81, below, with the Oracle home path.
@REM ---------------------------------------------------------------------------

@set ORACLE_HOME=D:\oracle\product\10.2.0\db_1

@REM ---------------------------------------------------------------------------
@REM Replace ora81, below, with the Oracle SID.
@REM ---------------------------------------------------------------------------

@set ORACLE_SID=your-sid

@REM ---------------------------------------------------------------------------
@REM Replace sys/manager, below, with the target connect string.
@REM ---------------------------------------------------------------------------

@set TARGET_CONNECT_STR=/

@REM ---------------------------------------------------------------------------
@REM Set the Oracle Recovery Manager.
@REM ---------------------------------------------------------------------------

@set RMAN=%ORACLE_HOME%\bin\rman.exe

@REM ---------------------------------------------------------------------------
@REM PROXY
@REM For a PROXY backup, uncomment the line below and replace the value.
@REM
@REM       NB_ORA_PC_STREAMS - specifies the number of parallel backup streams
@REM                           to be started.
@REM ---------------------------------------------------------------------------
@REM @set NB_ORA_PC_STREAMS=3


@REM ---------------------------------------------------------------------------
@REM Log the start of this scripts.
@REM ---------------------------------------------------------------------------

@for /F "tokens=1*" %%p in ('date /T') do @set DATE=%%p %%q
@for /F %%p in ('time /T') do @set DATE=%DATE% %%p

@echo ==== started on %DATE% ==== >> %RMAN_LOG_FILE%
@echo Script name: %0 >> %RMAN_LOG_FILE%

@REM ---------------------------------------------------------------------------
@REM Several RMAN commands use time parameters that require NLS_LANG and
@REM NLS_DATE_FORMAT to be set. This example uses the standard date format.
@REM Replace below with the desired language values.
@REM ---------------------------------------------------------------------------

@set NLS_LANG=american
@set NLS_DATE_FORMAT=YYYY-MM-DD:hh24:mi:ss

@REM ---------------------------------------------------------------------------
@REM Print out environment variables set in this script.
@REM ---------------------------------------------------------------------------

@echo #                                       >> %RMAN_LOG_FILE%
@echo   RMAN  :  %RMAN%                       >> %RMAN_LOG_FILE%
@echo   NLS_LANG  :  %NLS_LANG%               >> %RMAN_LOG_FILE%
@echo   ORACLE_HOME  :  %ORACLE_HOME%         >> %RMAN_LOG_FILE%
@echo   ORACLE_SID  :  %ORACLE_SID%           >> %RMAN_LOG_FILE%
@echo   NLS_DATE_FORMAT  :  %NLS_DATE_FORMAT% >> %RMAN_LOG_FILE%
@echo   RMAN_LOG_FILE  :  %RMAN_LOG_FILE%     >> %RMAN_LOG_FILE%

@REM ---------------------------------------------------------------------------
@REM PROXY
@REM For a PROXY backup, uncomment the line below.
@REM ---------------------------------------------------------------------------
@REM @echo   NB_ORA_PC_STREAMS  :  %NB_ORA_PC_STREAMS%     >> %RMAN_LOG_FILE%

@REM ---------------------------------------------------------------------------
@REM Print out environment variables set in bphdb.
@REM ---------------------------------------------------------------------------

@echo   NB_ORA_SERV  :  %NB_ORA_SERV%                     >> %RMAN_LOG_FILE%
@echo   NB_ORA_FULL  :  %NB_ORA_FULL%                     >> %RMAN_LOG_FILE%
@echo   NB_ORA_INCR  :  %NB_ORA_INCR%                     >> %RMAN_LOG_FILE%
@echo   NB_ORA_CINC  :  %NB_ORA_CINC%                     >> %RMAN_LOG_FILE%
@echo   NB_ORA_CLASS  :  %NB_ORA_CLASS%                   >> %RMAN_LOG_FILE%

@REM ---------------------------------------------------------------------------
@REM We assume that the database is properly opened. If desired, this would
@REM be the place to verify that.
@REM ---------------------------------------------------------------------------

@REM ---------------------------------------------------------------------------
@REM If this script is executed from a NetBackup schedule, NetBackup
@REM sets an NB_ORA environment variable based on the schedule type.
@REM For example, when:
@REM     schedule type is                BACKUP_TYPE is
@REM     ----------------                --------------
@REM Automatic Full                      INCREMENTAL LEVEL=0
@REM Automatic Differential Incremental  INCREMENTAL LEVEL=1
@REM Automatic Cumulative Incremental    INCREMENTAL LEVEL=1 CUMULATIVE
@REM
@REM For user initiated backups, BACKUP_TYPE defaults to incremental
@REM level 0 (Full).  To change the default for a user initiated
@REM backup to incremental or incrementatl cumulative, uncomment
@REM one of the following two lines.
@REM @set BACKUP_TYPE="INCREMENTAL LEVEL=1"
@REM @set BACKUP_TYPE="INCREMENTAL LEVEL=1 CUMULATIVE"
@REM
@REM Note that we use incremental level 0 to specify full backups.
@REM That is because, although they are identical in content, only
@REM the incremental level 0 backup can have incremental backups of
@REM level > 0 applied to it.
@REM ---------------------------------------------------------------------------

@REM ---------------------------------------------------------------------------
@REM What kind of backup will we perform.
@REM ---------------------------------------------------------------------------

@if "%NB_ORA_FULL%" EQU "1" @set BACKUP_TYPE=INCREMENTAL Level=0
@if "%NB_ORA_INCR%" EQU "1" @set BACKUP_TYPE=INCREMENTAL Level=1
@if "%NB_ORA_CINC%" EQU "1" @set BACKUP_TYPE=INCREMENTAL Level=1 CUMULATIVE
@if NOT DEFINED BACKUP_TYPE @set BACKUP_TYPE=INCREMENTAL Level=0

@REM ---------------------------------------------------------------------------
@REM Call Recovery Manager to initiate the backup. This example does not use a
@REM Recovery Catalog. If you choose to use one, remove the option, nocatalog,
@REM from the rman command line below and add a
@REM 'rcvcat <userid>/<passwd>@<tns alias>' statement.
@REM
@REM  NOTE WHEN USING TNS ALIAS: When connecting to a database
@REM  using a TNS alias, you must use a send command or a parms operand to
@REM  specify environment variables.  In other words, when accessing a database
@REM  through a listener, the environment variables set at the system level are not
@REM  visible when RMAN is running.  For more information on the environment
@REM  variables, please refer to the NetBackup for Oracle Admin. Guide.
@REM
@REM If you are getting an error that the input line is too long, you will need
@REM to put the RMAN run block in a separate file.  Then use the "cmdfile"
@REM option of RMAN.  For more information on the "cmdfile" options please
@REM refer to the RMAN documentation.
@REM ---------------------------------------------------------------------------

@REM ---------------------------------------------------------------------------
@REM PROXY
@REM For a PROXY backup, you must use a send command to specify
@REM the NB_ORA_PC_STREAMS environment variable. For example,
@REM echo ALLOCATE CHANNEL ch01 TYPE 'SBT_TAPE';
@REM echo SEND 'NB_ORA_PC_STREAMS=%%NB_ORA_PC_STREAMS%%';
@REM
@REM %BACKUP_TYPE% must also be removed and replaced with the PROXY parameter
@REM in the RMAN section associated with the data files.  For example,
@REM echo BACKUP
@REM echo       PROXY
@REM echo       FORMAT 'bk_u%%u_s%%s_p%%p_t%%t'
@REM echo       DATABASE;
@REM            .
@REM            .
@REM  Note that the controlfiles and archivelogs are not backed up using proxy
@REM  copy method. Rman will initiate non-proxy copy sessions to backup the
@REM  controlfile and archivelogs.
@REM ---------------------------------------------------------------------------
@(
echo RUN {
echo ALLOCATE CHANNEL ch00 TYPE 'SBT_TAPE';
echo ALLOCATE CHANNEL ch01 TYPE 'SBT_TAPE';
echo send 'NB_ORA_SERV=your_bk_server,NB_ORA_POLICY=your_policy';
echo BACKUP
echo       %BACKUP_TYPE%
echo       FORMAT 'bk_u%%u_s%%s_p%%p_t%%t'
echo       DATABASE;
echo sql 'alter system archive log current';
echo RELEASE CHANNEL ch00;
echo RELEASE CHANNEL ch01;
echo # Backup all archive logs
echo ALLOCATE CHANNEL ch00 TYPE 'SBT_TAPE';
echo BACKUP
echo       FILESPERSET 20
echo       FORMAT 'arch-s%%s-p%%p'
echo       ARCHIVELOG ALL;
echo RELEASE CHANNEL ch00;
echo ALLOCATE CHANNEL ch00 TYPE 'SBT_TAPE';
echo BACKUP
echo       FORMAT 'cntrl_%s_%p_%t'
echo       CURRENT CONTROLFILE;
echo RELEASE CHANNEL ch00;
echo }
) | %RMAN% target %TARGET_CONNECT_STR% nocatalog msglog '%RMAN_LOG_FILE%' append

@set ERRLEVEL=%ERRORLEVEL%

@REM ---------------------------------------------------------------------------
@REM NetBackup (bphdb) stores the name of a file in an environment variable,
@REM called STATUS_FILE. This file is used by an automatic schedule to
@REM communicate status information with NetBackup's job monitor. It is up to
@REM the script to write a 0 (passed) or 1 (failure) to the status file.
@REM ---------------------------------------------------------------------------

@if %ERRLEVEL% NEQ 0 @goto err

@set LOGMSG=ended successfully
@if "%STATUS_FILE%" EQU "" goto end
@echo 0 > "%STATUS_FILE%"
@goto end

:err
@set LOGMSG=ended in error
@if "%STATUS_FILE%" EQU "" @goto end
@echo 1 > "%STATUS_FILE%"

:end

@REM ---------------------------------------------------------------------------
@REM Log the completion of this script.
@REM ---------------------------------------------------------------------------

@for /F "tokens=1*" %%p in ('date /T') do @set DATE=%%p %%q
@for /F %%p in ('time /T') do @set DATE=%DATE% %%p

@echo #  >> %RMAN_LOG_FILE%
@echo %==== %LOGMSG% on %DATE% ==== >> %RMAN_LOG_FILE%
@endlocal
@REM End of Main Program -----------------------------------------------------
```
