---
title: A Brief Introduction of Oracle Role and Privilege
categories: oracle
comments: false
date: 2019-08-07 21:13:38
tags: fundamentals
---
This brief introduction will cover with the fundamentals of Oracle privilege as well as roles, including how to retrieve all privilege for a user, and how to grant necessary roles or privilege to a user.
<!--more-->

# 1. Basic Views of privileges and roles

* DBA_SYS_PRIVS
* DBA_TAB_PRIVS
* DBA_ROLE_PRIVS
* ROLE_SYS_PRIVS
* SESSION_PRIVS

___When granting a privilege to user, the PRIVILEGE will infect immediate; but when granting a role to a user, already connected session have to reconnect or use `set role all` to refresh the new granted role privileges___

# 2. Script to find all privileges

```sql
select privilege
  from dba_sys_privs
 where grantee = upper('&1')
union
select privilege
  from dba_sys_privs
 where grantee in
       (select granted_role from dba_role_privs where grantee = upper('&1'));
```

# 3. Analysis of specific roles

Roles privileges can be retrieved from `role_sys_privs` view:

___Resource role does not have create view privilege, but have unlimited tablespace___

```sql
sys@LINORA> select PRIVILEGE, role from role_sys_privs where role = upper('&1');
Enter value for 1: connect
old   1: select PRIVILEGE, role from role_sys_privs where role = upper('&1')
new   1: select PRIVILEGE, role from role_sys_privs where role = upper('connect')

PRIVILEGE			 ROLE
--------------------- ----------------------
SET CONTAINER			 CONNECT
CREATE SESSION			 CONNECT

-- resource role
Enter value for 1: resource
old   1: select PRIVILEGE, role from role_sys_privs where role = upper('&1')
new   1: select PRIVILEGE, role from role_sys_privs where role = upper('resource')

PRIVILEGE			 ROLE
------------------ --------------------------------------------------
CREATE TRIGGER			 RESOURCE
CREATE SEQUENCE 		 RESOURCE
CREATE TYPE			 RESOURCE
CREATE PROCEDURE		 RESOURCE
CREATE CLUSTER			 RESOURCE
CREATE OPERATOR 		 RESOURCE
CREATE INDEXTYPE		 RESOURCE
CREATE TABLE			 RESOURCE

-- olap user role
Enter value for 1: olap_user
old   1: select PRIVILEGE, role from role_sys_privs where role = upper('&1')
new   1: select PRIVILEGE, role from role_sys_privs where role = upper('olap_user')

PRIVILEGE					ROLE
------------------------ --------------------------------------------------
CREATE CUBE BUILD PROCESS	 OLAP_USER
CREATE CUBE DIMENSION	 	 OLAP_USER
CREATE MEASURE FOLDER	 	 OLAP_USER
CREATE JOB		 OLAP_USER
CREATE CUBE		 OLAP_USER
CREATE SEQUENCE	 OLAP_USER
CREATE VIEW		 OLAP_USER
CREATE TABLE		 OLAP_USER
```

Based on above information, we can grant application user `OLAP_USER`, `CONNECT` and `RESOURCE` roles.


# 4. Identify users with DBA role

```sql
set line 200
col grantee for a30
select * from dba_role_privs where granted_role='DBA';
```

__EOF__
