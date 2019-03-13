---
title: Security Hardening of VPS
categories: oracle
comments: false
date: 2019-03-13 12:23:59
tags:
---

For the perspective of security, Linux VPS should do some action for security hardening. Some of rules are also applicable for Linux server. This post based on CentOS 7.

<!--more-->
# 1. SSH configuration
Modify `/etc/ssh/sshd_config`, to:

* 1.1 Disable root ssh login

Disabled root log from ssh, you must create a normal user before do that.
```
PermitRootLogin no
```

* 1.2 Modify default ssh port

```
Port xxx    # from 0 ~ 65535
```
But don't use any port from range 0 to 1024, most of them are known port for other important service, such as HTTP 80 port.

* 1.3 Disable password authentication

Using RSA public keys instead:
```
# generate public rsa key from local server
ssh-keygen -t rsa

# copy public key to remote server
ssh-copy-id -i ~/.ssh/id_rsa.pub <Your Username for remote server>@<Your IP or Hostname> -p <Your SSH port>
```

# 2. Firewall configuration
Add ssh port to firewall whitelist.
```
# Install semanage first, for selinux policy management
yum provides /usr/sbin/semanage
# Tell selinux the new port
semanage port -a -t ssh_port_t -p tcp <PORT_NUM>

# Add whitelist to the firewall
firewall-cmd --permanent --zone=public --add-port=<PORT_NUM>/tcp
firewall-cmd --add-port <PORT_NUM>/tcp

# Reload firewall
firewall-cmd --reload

# Restart sevice and check the result
systemctl restart sshd.service
semanage port -l | grep ssh
ss -tnlp | grep ssh
```

# 3. Other configuration

* Generate random strong password:

```
openssl rand -base64 10
# or
openssl rand -hex 10
```

* Disable ping

```
vi /etc/sysctl.conf
net.ipv4.icmp_echo_ignore_all=1

sysctl -p
```



__EOF__
