---
layout: post
title: "Oracle Enterprise Management failed with locale failed"
date: 2014-04-29 14:29:37
comments: false
categories: oracle
tags: em
keywords: em, Enterprise Management,UNIX,Locale
description: emctl invoke perl warning:Setting locale failed
---
<h3>1.SYMPTOMS</h3>
调用<code>emctl</code>发生以下错误：
<!--more-->
```
scc:oracle $emctl status dbconsole
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
        LC_ALL = (unset),
        LC__FASTMSG = "true",
        LC_MESSAGES = "",
        LANG = "en_UN"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
OC4J Configuration issue. /u01/app/oracle/db_1/oc4j/j2ee/OC4J_DBConsole_scc_qffh not found.
```
<h3>2.CAUSE</h3>
跟用户环境变量lang设置有关，这个变量设置不正确导致。oracle用户环境变量中：<code>export NLS_LANG=American_America.ZHS16GBK </code>，怀疑OS没安装中文包。
<h3>3.SOLUTION</h3>
<code>exoport LC_ALL=C</code>
"C"是指 US7ASCII，这意味着仅可显示 a-z、A-Z 和 0-9。
注意上述报错，<code>OC4J Configuration issue. /u01/app/oracle/db_1/oc4j/j2ee/OC4J_DBConsole_scc_qffh not found.</code>，怀疑修改过主机名或者IP。
通过emca重新创建EM：
```
emca -deconfig dbcontrol db -repos drop
emca -config dbcontrol db -repos create
```
create的时候又报错：
```
SEVERE: Failed to allocate port(s) in the specified range(s) for the following process(es): JMS [5540-5559],RMI [5520-5539],Database Control [5500-5519],EM Agent [3938] | [1830-1849]
Refer to the log file at /u01/app/oracle/db_1/cfgtoollogs/emca/qffh/emca_2014-04-28_04-50-05-PM.log for more details.
```
通过指定端口，终于重新创建EM，并且能登录管理：
```
emca -config dbcontrol db -repos create -DBCONTROL_HTTP_PORT 5508 -AGENT_PORT 3940 -RMI_PORT 5524 -JMS_PORT 5545
```
<h3>4.EXPLANATION</h3>
对于Linux/UNIX下locale的解析：
在linux/UNIX上Locales用来定义用户所使用的语言。因为locales还定义了用户使用的字符集，所以，当语言中含有非ASCIIA字符时，设定好正确的locale就显得非常重要了。Locales是用以下的格式来定义的：
```
<lang>_<territory>.<codeset>[@<modifiers>]
```
要查看当前设置，请使用 “locale” 命令，如下所示：
<code>$ locale</code>
输出示例：
```
[root@oel1:/root]# locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```
Oracle建议，local尽量使用uft-8。
如果要查看所有安装了的locale，可以-a的参数，以下为部分输出：
```
[root@oel1:/root]# locale -a
zh_CN
zh_CN.gb18030
zh_CN.gb2312
zh_CN.gbk
zh_CN.utf8
zh_HK
zh_HK.big5hkscs
zh_HK.utf8
zh_SG
zh_SG.gb2312
zh_SG.gbk
zh_SG.utf8
zh_TW
zh_TW.big5
zh_TW.euctw
zh_TW.utf8
```
针对环境变量 <code>NLS_LANG</code> :
<code>NLS_LANG</code> 的构成为：<code>NLS_LANG=<NLS_LANGUAGE>_<NLS_TERRITORY>.<clients characterset></code>
以 locale "en_US.UTF-8"为例，这意味着应该将 <code>NLS_LANG</code> 设置为 <code>=AMERICAN_AMERICA.AL32UTF8</code>；如果 <code>locale</code> 设置为<code>fr_FR.UTF-8</code>，则相应的 <code>NLS_LANG</code> 设置将为 <code>FRENCH_FRANCE.AL32UTF8</code>。
<code>Locale</code> 和 <code>NLS_LANG</code> 设置（如果适用，还包括 telnet/ssh config）需要互相匹配，但从技术角度看它们均与数据库字符集无关，并且它们仅针对该客户端环境相关。
参照：
[International Language Environment Guide](http://docs.oracle.com/cd/E23824_01/html/E26033/glmbx.html)
[1548858.1](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=853836115233131&id=1548858.1&_afrWindowMode=0&_adf.ctrl-state=w2woxsr5c_81)
[1602518.1](https://support.oracle.com/epmos/faces/SearchDocDisplay?_adf.ctrl-state=w2woxsr5c_9&_afrLoop=853966831115856)
