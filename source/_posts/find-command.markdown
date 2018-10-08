---
layout: post
title: "Linux Find Command"
date: 2014-05-19 15:31:15
comments: false
categories: linux
tags: find
keywords: linux,find
description: find command in linux or UNIX
---
<p>Find在Linux/UNIX中，通过指定某个或者多个目录查找符合执行条件的文件，显示文件名或对这些文件进行特定的操作。</p>
<!--more-->

|表达式|参数|含义|
|----|----|----|
|-type | f/d | 查找目标的类型(文件/目录)|
|-size | +n/-n/n  | 文件大小超过/小于/等于n blocks|
|-mtime | +/-x  | x天以前/x天以内被修改的文件|
|-perm|mode|访问指定权限的文件(rwx)|
|-user|User|属于用户user的文件|
|-o||逻辑'or'|

### 1.查找文件名为oracle的文件，并打印在屏幕上
```
# ls -l /oracle
total 2580304
-rw-r-----    1 root     system   1321110528 May 16 18:30 10gr2_aix5l64_database.cpio
drwxrwxr-x    4 oracle   oinstall        256 May 16 17:19 app
drwxrwxr-x    2 oracle   oinstall        256 May 16 14:45 lost+found
-rw-r--r--    1 root     system            0 May 19 19:03 oracle
# find / -type f -name oracle -print
/oracle/oracle
# find / -type d -name oracle -print
/home/oracle
/oracle
/oracle/app/oracle
```

### 2.查找名为oracle的东西，且通过<code>ls -l</code>展示出来
```
# find / -name "oracle" -exec ls -l {} \;
total 16
drwxr-xr-x    3 oracle   oinstall        256 May 16 17:19 .java
-rwxr-----    1 oracle   oinstall        857 May 16 18:15 .profile
-rw-------    1 oracle   oinstall        794 May 16 18:15 .sh_history
total 2580304
-rw-r-----    1 root     system   1321110528 May 16 18:30 10gr2_aix5l64_database.cpio
drwxrwxr-x    4 oracle   oinstall        256 May 16 17:19 app
drwxrwxr-x    2 oracle   oinstall        256 May 16 14:45 lost+found
-rw-r--r--    1 root     system            0 May 19 19:03 oracle
total 0
drwxrwxr-x    4 oracle   oinstall        256 May 16 18:15 product
-rw-r--r--    1 root     system            0 May 19 19:03 /oracle/oracle
```
注意在例1中，以oracle为名的有三个目录，因此，例2中是列出了三个目录下的内容。
```
# ls -l /home/oracle
total 16
drwxr-xr-x    3 oracle   oinstall        256 May 16 17:19 .java
-rwxr-----    1 oracle   oinstall        857 May 16 18:15 .profile
-rw-------    1 oracle   oinstall        794 May 16 18:15 .sh_history
# ls -l /oracle
total 2580304
-rw-r-----    1 root     system   1321110528 May 16 18:30 10gr2_aix5l64_database.cpio
drwxrwxr-x    4 oracle   oinstall        256 May 16 17:19 app
drwxrwxr-x    2 oracle   oinstall        256 May 16 14:45 lost+found
-rw-r--r--    1 root     system            0 May 19 19:03 oracle
# ls -l /oracle/app/oracle
total 0
drwxrwxr-x    4 oracle   oinstall        256 May 16 18:15 product
```
### 3.查找权限为777，且修改时间为四天内的文件
```
# find . -perm 777 -mtime -4 -print
./oracle
```
### 4.删除N天前的文件
```
find . -type f -mtime +14 -exec rm -f {} \;
```
将exec换成ok，表示每删除一次确定一次，以防止误操作。
```
find . -type f -mtime +14 -ok rm -f {} \;
```
### 5.删除7天前所有名为a.out或者*.o的文件，且这些文件不是以nfs形式挂载的
```
find / \( -name a.out -o -name '*.o' \) -atime +7 ! -fstype nfs -exec rm -rf {} \;
```
### 6.查找当前目录下，以"*.trc"结尾，且文件内容含有"error"字样的行，打印出来
```
# find ./ -name "*.trc" -exec grep -i "error" '{}' \; -print
error: "OUT_CFS_LINK             " exit=$D1 error=$D2 svp=$D3 tdvp=$D4 name=$D5
./alter.trc
```
### 7.查找5个最大/最小的文件
```
# find . -type f -exec ls -s {} \; | sort -n -r | head -5
1290152 ./10gr2_aix5l64_database.cpio
114656 ./Disk1/stage/Components/oracle.rsf.hybrid/10.2.0.0.0/1/DataFiles/group.jar
94456 ./Disk1/stage/Components/oracle.rsf.hybrid/10.2.0.0.0/1/DataFiles/group_new.jar
93168 ./Disk1/stage/Components/oracle.rdbms.install.seeddb/10.2.0.1.0/1//Seed_Database.dfb
44868 ./Disk1/stage/Components/oracle.sysman.console.db/10.2.0.1.0/1/DataFiles/filegroup8.jar
# find . -type f -exec ls -s {} \; | sort -n  | head -5
   0 ./oracle
   4 ./Disk1/doc/dcommon/gifs/bookbig.gif
   4 ./Disk1/doc/dcommon/gifs/bookicon.gif
   4 ./Disk1/doc/dcommon/gifs/booklist.gif
   4 ./Disk1/doc/dcommon/gifs/contbig.gif
```
对于windows server而言，Windows 2003 R2 SP2以上的版本中，自带有forfiles命令，功能跟find类型，目前win7也有带这个工具，forfiles的用法如下：
在win2003 server有forfiles.exe命令刪除n天前的文件：
```
C:\WINDOWS\system32\forfiles /P E:\logbak\ /S /M *.gz /D -8 /C "cmd /c del @file"
FORFILES [/P pathname] [/M searchmask] [/S] [/C command] [/D [+ | -] {yyyy-MM-dd | dd}]
/P pathname表示開始搜索的路徑，deault為當前目錄
/M serchmask 根據搜索掩碼搜索文件
/S 指導forfiles遞歸到子目錄
/C command表示為每個file執行的命令，命令字符串應該用雙引號括起來
/D date 選擇文件，其上一次時間大于或扽gui(+)，或者小于或等于(—)，用"yyyy-MM-dd"格式指定的日期；
	或者選擇文件，其上一次修改日期大于或等于(+)當前日期加"dd"天，或者小于等于(-)當前日期減"dd"天
	默認命令是"cmd /c echo @file"，以下變量可以用在命令字符串中：
		@file	 --返回文件名
		@fname	 --返回不帶擴展名的文件名
```


find命令好像不能删除某段时间的问题，比如删除2012年的文件，可以尝试用如下脚本：

```
for filename in /home/oracle/other/*;
do if [ `date -r $filename +%Y` == "2012" ]
then rm -f $filename
fi done
```




