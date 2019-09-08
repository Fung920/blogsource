---
title: Security Hardening of Oracle Database
categories: oracle
comments: false
date: 2019-08-06 11:03:23
tags: how to
---
This post introduces how to implement security hardening of Oracle database.
<!--more-->

## 1. Verification function of password limits

Below script contains a verification function name `ora12c_verify_function`:
```sql
[oracle@cdb01 ~]$ ls -l $ORACLE_HOME/rdbms/admin/utlpw*
-rw-r--r--. 1 oracle oinstall 12543 11月  7 2013 /oracle/app/oracle/product/12c/db_1/rdbms/admin/utlpwdmg.sql
```

## 2. Resource limit of default profile after executing utlpwdmg.sql

utlpwdmg.sql script will modify default profile as below rules:

```sql
ALTER PROFILE DEFAULT LIMIT
PASSWORD_LIFE_TIME 180
PASSWORD_GRACE_TIME 7
PASSWORD_REUSE_TIME UNLIMITED
PASSWORD_REUSE_MAX  UNLIMITED
FAILED_LOGIN_ATTEMPTS 10
PASSWORD_LOCK_TIME 1
PASSWORD_VERIFY_FUNCTION ora12c_verify_function;
```

Also create `ORA_STIG_PROFILE` as below rules:

```
SQL> col profile for a30
col resource for a30
col limit for a50
set line 200 pagesize 200
select * from dba_profiles;
SQL> 
PROFILE 		       RESOURCE_NAME			RESOURCE LIMIT						    COM
------------------------------ -------------------------------- -------- -------------------------------------------------- ---
DEFAULT 		       COMPOSITE_LIMIT			KERNEL	 UNLIMITED					    NO
DEFAULT 		       SESSIONS_PER_USER		KERNEL	 UNLIMITED					    NO
DEFAULT 		       CPU_PER_SESSION			KERNEL	 UNLIMITED					    NO
DEFAULT 		       CPU_PER_CALL			KERNEL	 UNLIMITED					    NO
DEFAULT 		       LOGICAL_READS_PER_SESSION	KERNEL	 UNLIMITED					    NO
DEFAULT 		       LOGICAL_READS_PER_CALL		KERNEL	 UNLIMITED					    NO
DEFAULT 		       IDLE_TIME			KERNEL	 UNLIMITED					    NO
DEFAULT 		       CONNECT_TIME			KERNEL	 UNLIMITED					    NO
DEFAULT 		       PRIVATE_SGA			KERNEL	 UNLIMITED					    NO
DEFAULT 		       FAILED_LOGIN_ATTEMPTS		PASSWORD 10						    NO
DEFAULT 		       PASSWORD_LIFE_TIME		PASSWORD 180						    NO
DEFAULT 		       PASSWORD_REUSE_TIME		PASSWORD UNLIMITED					    NO
DEFAULT 		       PASSWORD_REUSE_MAX		PASSWORD UNLIMITED					    NO
DEFAULT 		       PASSWORD_VERIFY_FUNCTION 	PASSWORD NULL						    NO
DEFAULT 		       PASSWORD_LOCK_TIME		PASSWORD 1						    NO
DEFAULT 		       PASSWORD_GRACE_TIME		PASSWORD 7						    NO
ORA_STIG_PROFILE	       COMPOSITE_LIMIT			KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       SESSIONS_PER_USER		KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       CPU_PER_SESSION			KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       CPU_PER_CALL			KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       LOGICAL_READS_PER_SESSION	KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       LOGICAL_READS_PER_CALL		KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       IDLE_TIME			KERNEL	 15						    NO
ORA_STIG_PROFILE	       CONNECT_TIME			KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       PRIVATE_SGA			KERNEL	 DEFAULT					    NO
ORA_STIG_PROFILE	       FAILED_LOGIN_ATTEMPTS		PASSWORD 3						    NO
ORA_STIG_PROFILE	       PASSWORD_LIFE_TIME		PASSWORD 60						    NO
ORA_STIG_PROFILE	       PASSWORD_REUSE_TIME		PASSWORD 365						    NO
ORA_STIG_PROFILE	       PASSWORD_REUSE_MAX		PASSWORD 10						    NO
ORA_STIG_PROFILE	       PASSWORD_VERIFY_FUNCTION 	PASSWORD ORA12C_STRONG_VERIFY_FUNCTION			    NO
ORA_STIG_PROFILE	       PASSWORD_LOCK_TIME		PASSWORD UNLIMITED					    NO
ORA_STIG_PROFILE	       PASSWORD_GRACE_TIME		PASSWORD 5						    NO
```


## 3. Difference between 11g and 12c

### 3.1 verify_function_11G Function Password Requirements

This function checks for the following requirements when users create or modify passwords:
   - The password is not the same as the user name, nor is it the user name spelled backward or with the numbers 1–100 appended.
   - The password is not the same as the server name or the server name with the numbers 1–100 appended.
   - The password is not too simple (for example, oracle, oracle with the numbers 1–100 appended, welcome1, database1, account1, user1234, password1, oracle123, computer1, abcdefg1, or change_on_install).
   - The password includes at least 1 numeric and 1 alphabetic character.
   - The password differs from the previous password by at least 3 characters.

The following internal checks are also applied:
   - The password contains no fewer than 8 characters and does not exceed 30 characters.
   - The password does not contain the double-quotation character ("). It can be surrounded by double-quotation marks, however.

### 3.2 ora12c_verify_function Password Requirements

The ora12c_verify_function function provides requirements that the Department of Defense Database Security Technical Implementation Guide recommends.

This function checks for the following requirements when users create or modify passwords:
   * The password contains no fewer than 8 characters and includes at least 1 numeric and 1 alphabetic character.
   * The password is not the same as the user name or the user name reversed.
   * The password is not the same as the database name.
   * The password does not contain the word oracle (such as oracle123).
   * The password is not too simple (for example, welcome1, database1, account1, user1234, password1, oracle123, computer1, abcdefg1, or change_on_install).
   * The password differs from the previous password by at least 3 characters.
   * The password contains at least one special character.

The following internal checks are also applied:
   * The password does not exceed 30 characters.
   * The password does not contain the double-quotation character ("). It can be surrounded by double-quotation marks, however.


### 3.3 ora12c_strong_verify_function Function Password Requirements
The ora12c_strong_verify_function function fulfills the Department of Defense Database Security Technical Implementation Guide requirements.

This function checks for the following requirements when users create or modify passwords:

   * The password must contain at least 2 upper case characters, 2 lower case characters, 2 numeric characters, and 2 special characters. These special characters are as follows:
       ```
       ‘ ~ ! @ # $ % ^ & * ( ) _ - + = { } [ ] \ / < > , . ; ? ' : | (space)
       ```
   * The password must differ from the previous password by at least 4 characters.

The following internal checks are also applied:
   * The password contains no fewer than nine characters and does not exceed 30 characters.
   * The password does not contain the double-quotation character ("). It can be surrounded by double-quotation marks, however.



## 4. Example of modifying user profile

```sql
alter profile default limit PASSWORD_LIFE_TIME UNLIMITED;
alter profile default limit PASSWORD_VERIFY_FUNCTION null;

```

## 5. Summary
Even with Oracle 12c, security hardening is not enabled by force, DBA need to execute `utlpwdmg.sql` manually to enable.
By default, this script will update default profile, which is the default profile for all users. ___It's recommended to modify this script, not using default profile for all users___.


Reference:
[Database Security Guide-Configuring Authentication](https://docs.oracle.com/database/121/DBSEG/authentication.htm#DBSEG3225)

__EOF__
