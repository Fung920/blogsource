---
layout: post
title: "Setup DNS Server in RHEL6"
date: 2016-04-18 17:35:34
comments: false
categories: linux
tags: dns
keywords: DNS server, RHEL6
description:  How to setup DNS server in RHEL6 by using BIND
---
In the previous post [11gr2 RAC SCAN DNS Configuration](/11gr2-scan-dns.html) describes 11gr2 SCAN concepts and how to setup SCAN DNS in rhel5. But as I upgraded the OS to version RHEL6, found something different wiht RHEL5, this post is explain how to setup primary and slave DNS server in RHEL6.
<!--more-->

Below is my testing environment:

``` bash
192.168.56.101	node1		#primary DNS server
192.168.56.102	node2		#slave DNS server
192.168.56.103	node1-vip	#below are node machines IP address
192.168.56.104	node2-vip
192.168.56.110 racdb-scan
192.168.56.111 racdb-scan
192.168.56.112 racdb-scan
```

### 1. Primary server setup steps
#### 1.1 Install necessary rpms

``` bash
[root@node1 ~]# yum install bind* -y
```

#### 1.2 Configure named daemon file

``` bash
[root@node1 ~]# vi /etc/named.conf
[root@node1 ~]# cat /etc/named.conf
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
	listen-on port 53 { 127.0.0.1; 192.168.56.101; };  //Master DNS server IP addr
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-query     { localhost; 192.168.56.0/24; };	//define the IP range which can be resolved
    allow-transfer  { localhost; 192.168.56.102; }; 		// Slave DNS Servers IP
	recursion yes;

	dnssec-enable yes;
	dnssec-validation yes;
	dnssec-lookaside auto;

	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.iscdlv.key";

	managed-keys-directory "/var/named/dynamic";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

zone "oraclema.com" IN {   		//add forward zone file
        type master;
        file "node1.oraclema.zero";  	//this zone file should be located at /var/named/
        allow-update { none; };
};

zone "56.168.192.in-addr.arpa" IN {   //add reserve zone file
        type master;
        file "56.168.192.local";   	//reserve zone file also located at /var/named/
        allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

#### 1.4 Configure the forward and reserve zone file

``` bash
[root@node1 ~]# cd /var/named/
[root@node1 ~]# cp /var/named/named.localhost /var/named/node1.oraclema.zero
[root@node1 ~]# cp /var/named/named.localhost /var/named/56.168.192.local
[root@node1 named]# ls -ltr
...
-rw-r----- 1 root  named 1345 Apr 18 17:22 node1.oraclema.zero
-rw-r----- 1 root  named 1099 Apr 18 17:24 56.168.192.local
[root@node1 named]# cat node1.oraclema.zero
$TTL    86400
@               IN SOA  node1.oraclema.com.      root.oraclema.com. (
                                        42              ; serial (d. adams)
                                        3H              ; refresh
                                        15M             ; retry
                                        1W              ; expiry
                                        1D )            ; minimum
@        IN      NS      node1.oraclema.com.
@        IN      NS      node2.oraclema.com.
@	IN	A	192.168.56.110
@       IN      A       192.168.56.111
@       IN      A       192.168.56.112
@       IN      A       192.168.56.101
@       IN      A       192.168.56.102
@       IN      A       192.168.56.103
@       IN      A       192.168.56.104
racdb-scan                       IN A     192.168.56.110
racdb-scan                       IN A     192.168.56.111
racdb-scan                       IN A     192.168.56.112
node1-vip                        IN A     192.168.56.103
node2-vip                        IN A     192.168.56.104
node1                            IN A     192.168.56.101
node2                            IN A     192.168.56.102

[root@node1 named]# cat 56.168.192.local
$TTL    86400
@       IN      SOA     node1.oraclema.com. root.oraclema.com.  (
                                      1997022700 ; Serial
                                      28800      ; Refresh
                                      14400      ; Retry
                                      3600000    ; Expire
                                      86400 )    ; Minimum
@        IN      NS      node1.oraclema.com.
@        IN      NS      node2.oraclema.com.
@        IN      PTR	oraclema.com
node1	IN	A	192.68.56.101
node2	IN	A	192.168.56.102
node1-vip	IN	A	192.168.56.103
node2-vip	IN	A	192.168.56.104
racdb-scan	IN	A	192.168.56.110
racdb-scan      IN      A       192.168.56.111
racdb-scan      IN      A       192.168.56.112
101      IN      PTR     node1.oraclema.com.
102      IN      PTR     node2.oraclema.com.
110      IN      PTR     racdb-scan.oraclema.com.
111      IN      PTR     racdb-scan.oraclema.com.
112      IN      PTR     racdb-scan.oraclema.com.
102      IN      PTR     node1-vip.oraclema.com.
103      IN      PTR     node2-vip.oraclema.com.
```

After edited the forward and reserve files, run a error test.

```
[root@node1 named]# named-checkconf /etc/named.conf
[root@node1 named]# named-checkzone oraclema.com ./node1.oraclema.zero
zone oraclema.com/IN: loaded serial 42
OK
[root@node1 named]# named-checkzone 56.168.192.in-addr.arpa ./56.168.192.local
zone 56.168.192.in-addr.arpa/IN: loaded serial 1997022700
OK
```

If no errors, the named process can be started now:

```
[root@node1 named]# service named restart
Stopping named: .                                          [  OK  ]
Starting named:                                            [  OK  ]
[root@node1 named]# chkconfig named on
```

Finally, take a look at <code>dig</code> and <code>nslookup</code> command to check whether can resolve the domain or not:

```
[root@node1 named]# dig node1.oraclema.com

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> node1.oraclema.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56422
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1

;; QUESTION SECTION:
;node1.oraclema.com.		IN	A

;; ANSWER SECTION:
node1.oraclema.com.	86400	IN	A	192.168.56.101

;; AUTHORITY SECTION:
oraclema.com.		86400	IN	NS	node2.oraclema.com.
oraclema.com.		86400	IN	NS	node1.oraclema.com.

;; ADDITIONAL SECTION:
node2.oraclema.com.	86400	IN	A	192.168.56.102

;; Query time: 1 msec
;; SERVER: 192.168.56.101# 53(192.168.56.101)
;; WHEN: Mon Apr 18 18:02:13 2016
;; MSG SIZE  rcvd: 102

[root@node1 named]# nslookup node1.oraclema.com
Server:		192.168.56.101
Address:	192.168.56.101# 53

Name:	node1.oraclema.com
Address: 192.168.56.101
```

### 2. Slave DNS server setup steps
#### 2.1 Install necessary rpms
```
[root@node1 ~]# yum install bind* -y
```

#### 2.2 Configure named daemon file

``` bash
[root@node2 ~]# cat /etc/named.conf
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
	listen-on port 53 { 127.0.0.1; 192.168.56.102; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-query     { localhost; 192.168.56.0/24; };	//IP address range
	recursion yes;

	dnssec-enable yes;
	dnssec-validation yes;
	dnssec-lookaside auto;

	/* Path to ISC DLV key */
	bindkeys-file "/etc/named.iscdlv.key";

	managed-keys-directory "/var/named/dynamic";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};

zone "oraclema.com" IN {   			//define forward zone file
        type slave;
        file "slaves/node1.oraclema.zero";
        masters { 192.168.56.101; };
};

zone "56.168.192.in-addr.arpa" IN {   		//define reserve zone file
        type slave;
        file "slaves/56.168.192.local";
        masters { 192.168.56.101; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

After edit the named.conf file, no need to create or edit the zone files, when named process start, these files would be created automatically.

```
[root@node2 ~]# service named restart
Stopping named:                                            [  OK  ]
Starting named:                                            [  OK  ]
[root@node2 ~]# chkconfig named on
```

The two zone files are generated automatically by named process:

```
[root@node2 ~]# ls -l /var/named/slaves/
total 8
-rw-r--r-- 1 named named 720 Apr  8 21:53 56.168.192.local
-rw-r--r-- 1 named named 691 Apr 18 16:43 node1.oraclema.zero
```

#### 2.3 Check the resolve result

```
[root@node2 ~]# dig node2.oraclema.com

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> node2.oraclema.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9104
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1

;; QUESTION SECTION:
;node2.oraclema.com.		IN	A

;; ANSWER SECTION:
node2.oraclema.com.	86400	IN	A	192.168.56.102

;; AUTHORITY SECTION:
oraclema.com.		86400	IN	NS	node1.oraclema.com.
oraclema.com.		86400	IN	NS	node2.oraclema.com.

;; ADDITIONAL SECTION:
node1.oraclema.com.	86400	IN	A	192.168.56.101

;; Query time: 1 msec
;; SERVER: 192.168.56.101# 53(192.168.56.101)
;; WHEN: Mon Apr 18 19:05:23 2016
;; MSG SIZE  rcvd: 102
[root@node2 ~]# nslookup node1.oraclema.com
Server:		192.168.56.101
Address:	192.168.56.101# 53

Name:	node1.oraclema.com
Address: 192.168.56.101

[root@node2 ~]# nslookup 192.168.56.102
Server:		192.168.56.101
Address:	192.168.56.101# 53

102.56.168.192.in-addr.arpa	name = node1-vip.oraclema.com.
102.56.168.192.in-addr.arpa	name = node2.oraclema.com.
102.56.168.192.in-addr.arpa	name = node2.
102.56.168.192.in-addr.arpa	name = node1-vip.
```

### 3. Client setting

For this scenario, the node1 and node2 both can be treated as clients.

#### 3.1 Edit the hostname file

``` bash [Edit the hostname]
[root@node1 ~]# cat /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=node1.oraclema.com
GATEWAY=192.168.56.1
NOZEROCONF=yes
[root@node1 ~]# hostname
node1.oraclema.com
```
No need to manually change hostname to node1.oraclema.com, when named services is effective, the hostname can turn to node1.oraclema.com automatically.

#### 3.2 Edit Network Interface Card setting

This step can generate /etc/resolve.conf file as specified automatically.  Adding following line to NIC configuration files, for example, /etc/sysconfig/network-scripts/ifcfg-eth0 .

``` bash [Edit NIC Configuration]
DNS1=192.168.56.101
DNS2=192.168.56.102
DOMAIN=oraclema.com
```

#### 3.3 Change the resolve order

This step grantee the hostname resolve by DNS instead of hosts files /etc/hosts.

``` bash
[root@node1 ~]# cat /etc/host.conf
order bind,hosts
multi on
[root@node1 ~]# grep -i dns /etc/nsswitch.conf |grep -v ^#
hosts:      dns files
```

That's it, I can setup my SCAN in Oracle RAC environment now.

</br><b>EOF</b></br>
