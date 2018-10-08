---
layout: post
title: "Archiving and Compressing Files"
date: 2014-05-19 17:20:54 +0800
comments: false
categories: linux
tags: tar
keywords: tar,cpio,linux or unix archive and compress files
description: tar,cpio,linux or unix archive and compress files
---
一、在linux或者unix环境，DBA或者SA经常要对文档归档压缩以保存至远程机器。在linux/unix环境用的比较多的命令是<code>tar</code>、<code>cpio</code>和<code>zip</code>。以下是几个命令的几个简单用法。
<!--more-->
1.<code>tar</code>命令
```
$tar –cvf tarfilename.tar source_file(or source diretory)
```
c表示创建一个tar文件，f表示tar包名称，f后面必须紧接tar包名称，否则会出现错误。
如需压缩，则用z(gzip)或者j(bzip2)参数：
```
$tar -cvzf orahome.tar.gz /oracle/product/10.2
```
如果使用的是非GNU的tar，则可能无-j或者-z参数(HP-UX 11.23遇到过),这时候想要压缩，则须通过pipe结合其他压缩命令达到此目的:
```
$ tar -cvf - /oracle/product/10.2 | gzip > orahome.tar.gz
```
同时，tar还可以cp某个目录至另外一个目录而不需要借助中间介质：
```
$ tar -cvf - scripts | (cd /ora01/backup; tar -xvf -)
```
上面给出的例子表示将当前目录下scripts目录cp至/ora01/bakcup下。
Tar包的解压命令只需要该c为x(extrate),如$ tar -xvf mytar.tar
2.<code>cpio</code>命令
<code>cpio</code>通常用参数-ov表示创建cpio文件，通常扩展名为.cpio。以下命令表示ls出来的结果pipe给cpio，创建文件名为backup.cpio的文档。
```
$ ls | cpio -ov > backup.cpio
```
如果需要打包整个目录怎么办？可以用find结合，如：
```
$ find . -depth | cpio -ov > orahome.cpio
```
如果需要压缩，则可与gzip结合：
```
$ find . -depth | cpio -ov | gzip > orahome.cpio.gz
```
以下是cpio打包的语法：
```
$ [find or ls command] | cpio -o[other options] > filename
```
<code>cpio</code>解包一般指定idmv参数(AIX中，还需要c参数)，i表示从cpio包重定向，d和m参数则表示在解包中会自动创建目录和保留文件修改时间。
```
$ cpio -idvm < linux10g_disk1.cpio
$ cat linux10g_disk1.cpio | cpio -idvm
$ cat linux10g_disk1.cpio.gz | gunzip | cpio –idvm
# cpio -idmv < 10gr2_aix5l64_database.cpio


 cpio: 0511-903 Out of phase!
         cpio attempting to continue...


 cpio: 0511-904 skipping 732944 bytes to get back in phase!
         One or more files lost and the previous file is possibly corrupt!

cpio: 0511-027 The file name length does not match the expected value.
#FOR AIX
       c
            Reads and writes header information in ASCII character form. If a cpio archive was created using the c flag, it must be extracted with c flag.
cpio -idcmv < 10gr2_aix5l64_database.cpio
```
当然，从cpio包中也可以只释放其中的一个文件，以下例子表示从dbascripts.cpio中释放出rman.bsh这个脚本：
```
$ cpio -idvm rman.bsh < dbascripts.cpio
```
二、在打包好的文件中添加文件
1.<code>tar</code>
往tar包中添加文件，加上参数-r(append)：
```
$ tar -rvf backup.tar newscript.sql
```
或添加目录：
```
$ tar -rvf backup.tar scripts
```
2.<code>cpio</code>
cpio添加-A(append),同时也会指定F参数指定添加文件到指定的cpio包中：
```
$ ls *.sql | cpio -ovAF my.cpio
```
目录：
```
$ find backup | cpio -ovAF my.cpio
```
3.<code>zip</code>
zip包添加-g参数：
```
$ zip -g my.zip script.sql
目录：
$ zip -gr my.zip backup
```
注意：在HP-UX或者AIX下，并不会自动安装<code>unzip</code>工具，此时，需要用到jar去解压：
```
# man unzip
Manual entry for unzip not found or not installed.
# jar -xvf p10404530_112030_AIX64-5L_2of7.zip
# jar -cvf database.zip database/
```
