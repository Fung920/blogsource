---
layout: post
title: "Multiple version Python on the same machine"
date: 2017-04-21 04:47:06
comments: false
categories: Python
tags: Python
keywords: Upgrade Python
description: Installing latest Python without disruptting current one
name:
author: Fung Kong
datePublished:
---
Python is a critical package on Linux system, if you removed the current version before you install the new version, the OS maybe crash. So the best practice of upgrade Python on RHEL is reserved previous one.
<!--more-->

## Installing latest version
I want to install the latest version for Python2 and Python3, download the source code package from official site, and extract them one by one.
**Extract the source code**

```bash
yum install openssl-devel zlib-devel -y
tar -xzvf Python-3.6.1.tgz
tar -xzvf Python-2.7.13.tgz
```

**Compile and install the python from source code**

```bash
cd Python-2.7.13
or
cd Python-3.6.1
./configure --enable-shared --prefix=/usr/local/
make
sudo make altinstall
```

**create the symbolic link for python2.7 or python3.6 library**

```bash
#On 64 bit OS
# ldd `python2.7` to check the dependency
sudo ln -s /usr/local/lib/libpython2.7.so.1.0 /usr/lib64/libpython2.7.so.1.0
#for python3.6
sudo ln -s /usr/local/lib/libpython3.6m.so.1.0 /usr/lib64/libpython3.6m.so.1.0
```

## Install pip for python2.7
pip on Python3 will be automatically installed, for python2.7, install `pip` from `get-pip.py` is recommended.

```bash
wget https://bootstrap.pypa.io/get-pip.py
python2.7 get-pip.py
```

Now, we have two version of pip.

## Trouble shooting

> pip is configured with locations that require TLS/SSL, however the ssl module in Python is not available.




Because we missed to install the openssl-devel package, install the package and re-compile/reinstall the source code will resolve this issue.

```bash
yum install openssl-devel zlib-devel -y
```

> python2.7: error while loading shared libraries: libpython2.7.so.1.0: cannot open shared object file: No such file or directory


Check the lib dependency and found that some libs are missing, create the symbolic link will resolve this issue.

```bash
$ ldd /usr/local/bin/python2.7
	linux-vdso.so.1 =>  (0x00007fffd91ff000)
	libpython2.7.so.1.0 => not found
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f46bf554000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f46bf350000)
	libutil.so.1 => /lib64/libutil.so.1 (0x00007f46bf14d000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f46beec8000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f46beb34000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f46bf77d000)
```

Creating the symbolic link for python library

```bash
#On 64 bit OS
# ldd `python2.7` to check the dependency
sudo ln -s /usr/local/lib/libpython2.7.so.1.0 /usr/lib64/libpython2.7.so.1.0
#for python3.6
sudo ln -s /usr/local/lib/libpython3.6m.so.1.0 /usr/lib64/libpython3.6m.so.1.0
```

## Hint for virtualenvwrapper

`virtualenvwrapper` for python virtual environment management, install this package from any version of pip, virtualenvwrapper can cover any version of python, such as:

**Specified version of python to create virtual environment**

```bash
# for python2
mkvirtualenv py2 --python=/usr/local/bin/python2.7
$ python -V
Python 2.7.13
$ pip -V
pip 9.0.1 from /home/kingsley/.virtualenvs/py2/lib/python2.7/site-packages (python 2.7)

# for python3
mkvirtualenv py2 --python=/usr/local/bin/python3.6
$ python -V
Python 3.6.1
$ pip -V
pip 9.0.1 from /home/kingsley/.virtualenvs/py2/lib/python3.6/site-packages (python 3.6)
```

**Deactivating virtual environment**

```bash
$ deactivate
```

**Activating virtual environment**

```bash
$ workon py2
```

**Removing virtual environment**

```bash
$ rmvirtualenv py2
Removing py2...
```




***EOF***
