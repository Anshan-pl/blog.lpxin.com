---
title: rhel7上配置Mysql 8
date: 2019-10-21 21:07:09
tags: [rhel,redhat,mysql,linux,shell]
categories: [Mysql]
---

<center>
基于redhat7系统上安装部署Mysql 8<br/>
并给出Mysql配置使用流程<br/>
该教程为之前Mysql使用回顾并结合Mysql 8新特性
</center>

<!--more-->

# Mysql8安装包下载解压
[直接放出地址](https://dev.mysql.com/downloads/mysql)
选择自己对应的版本下载，一般默认下载第一项。
我这里使用wget命令下载，接着解压安装即可：
```
[root@localhost ~]# wget https://cdn.mysql.com//Downloads/MySQL-8.0/mysql-8.0.18-1.el7.x86_64.rpm-bundle.tar
--2019-10-21 22:02:56--  https://cdn.mysql.com//Downloads/MySQL-8.0/mysql-8.0.18-1.el7.x86_64.rpm-bundle.tar
Resolving cdn.mysql.com (cdn.mysql.com)... 104.75.165.42
Connecting to cdn.mysql.com (cdn.mysql.com)|104.75.165.42|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 684851200 (653M) [application/x-tar]
Saving to: 'mysql-8.0.18-1.el7.x86_64.rpm-bundle.tar'

100%[==============================================================================>] 684,851,200 3.75MB/s   in 3m 10s

2019-10-21 22:06:07 (3.43 MB/s) - 'mysql-8.0.18-1.el7.x86_64.rpm-bundle.tar' saved [684851200/684851200]

[root@localhost ~]# tar -xvf mysql-8.0.18-1.el7.x86_64.rpm-bundle.tar -C /root/mysql
mysql-community-libs-8.0.18-1.el7.x86_64.rpm
mysql-community-devel-8.0.18-1.el7.x86_64.rpm
mysql-community-embedded-compat-8.0.18-1.el7.x86_64.rpm
mysql-community-libs-compat-8.0.18-1.el7.x86_64.rpm
mysql-community-common-8.0.18-1.el7.x86_64.rpm
mysql-community-test-8.0.18-1.el7.x86_64.rpm
mysql-community-server-8.0.18-1.el7.x86_64.rpm
mysql-community-client-8.0.18-1.el7.x86_64.rpm

[root@localhost ~]# cd mysql

[root@localhost mysql]# ls
mysql-community-client-8.0.18-1.el7.x86_64.rpm           mysql-community-libs-8.0.18-1.el7.x86_64.rpm
mysql-community-common-8.0.18-1.el7.x86_64.rpm           mysql-community-libs-compat-8.0.18-1.el7.x86_64.rpm
mysql-community-devel-8.0.18-1.el7.x86_64.rpm            mysql-community-server-8.0.18-1.el7.x86_64.rpm
mysql-community-embedded-compat-8.0.18-1.el7.x86_64.rpm  mysql-community-test-8.0.18-1.el7.x86_64.rpm
```

# Mysql 8安装
```
[root@localhost mysql]# rpm -ivh mysql-community-server*.rpm
warning: mysql-community-server-8.0.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
error: Failed dependencies:
        mysql-community-client(x86-64) >= 8.0.11 is needed by mysql-community-server-8.0.18-1.el7.x86_64
        mysql-community-common(x86-64) = 8.0.18-1.el7 is needed by mysql-community-server-8.0.18-1.el7.x86_64
        net-tools is needed by mysql-community-server-8.0.18-1.el7.x86_64
```
可以看到，这里直接安装mysql-server会提示缺少依赖：client、common和net-tools，我们先安装这三个依赖，我这里的rhel7是mini-install所以默认没有安装net-tools，正常情况下这个工具是安装好的。
```
[root@localhost mysql]# yum install net-tools -y
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package net-tools.x86_64 0:2.0-0.25.20131004git.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

========================================================================================================================
 Package                   Arch                   Version                                    Repository            Size
========================================================================================================================
Installing:
 net-tools                 x86_64                 2.0-0.25.20131004git.el7                   base                 306 k

Transaction Summary
========================================================================================================================
Install  1 Package

Total download size: 306 k
Installed size: 917 k
Downloading packages:
net-tools-2.0-0.25.20131004git.el7.x86_64.rpm                                                    | 306 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : net-tools-2.0-0.25.20131004git.el7.x86_64                                                            1/1
  Verifying  : net-tools-2.0-0.25.20131004git.el7.x86_64                                                            1/1

Installed:
  net-tools.x86_64 0:2.0-0.25.20131004git.el7

Complete!
#我这里因为没有net-tools所以先使用yum安装上，接下来继续安装mysql的两个依赖项：

[root@localhost mysql]# rpm -ivh mysql-community-client*.rpm mysql-community-common*.rpm
warning: mysql-community-client-8.0.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
error: Failed dependencies:
        mysql-community-libs(x86-64) >= 8.0.11 is needed by mysql-community-client-8.0.18-1.el7.x86_64
```
可以看到，这里还是报error，提示缺少libs，那么就先安装libs：
```
[root@localhost mysql]# rpm -ivh mysql-community-libs*.rpm
warning: mysql-community-libs-8.0.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
error: Failed dependencies:
        mysql-community-common(x86-64) >= 8.0.11 is needed by mysql-community-libs-8.0.18-1.el7.x86_64
        mariadb-libs is obsoleted by mysql-community-libs-8.0.18-1.el7.x86_64
        mariadb-libs is obsoleted by mysql-community-libs-compat-8.0.18-1.el7.x86_64
```
这里还是存在error，提示系统上存在过期的依赖项mariadb，这个mariadb是系统自带的数据库，我们先将其卸载：
`[root@localhost mysql]# rpm -e mariadb-libs --nodeps`
这里使用-e参数表示卸载软件包，加上--nodeps是强制卸载，卸载完成后继续安装libs，这里直接三个一起安装了：
```
[root@localhost mysql]# rpm -ivh mysql-community-client*.rpm mysql-community-common*.rpm mysql-community-libs*.rpm
warning: mysql-community-client-8.0.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-common-8.0.18-1.e################################# [ 25%]
   2:mysql-community-libs-8.0.18-1.el7################################# [ 50%]
   3:mysql-community-client-8.0.18-1.e################################# [ 75%]
   4:mysql-community-libs-compat-8.0.1################################# [100%]
```
mysql的全部依赖项已经全部安装完毕，继续安装mysql-server：
```
[root@localhost mysql]# rpm -ivh mysql-community-server*.rpm
warning: mysql-community-server-8.0.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-server-8.0.18-1.e################################# [100%]
```
mysql 8到这里就已经安装完成了。

# Mysql 8配置root用户
首先要启动mysql服务，并查看临时root密码：
```
[root@localhost mysql]# systemctl start mysqld

[root@localhost mysql]# cat /var/log/mysqld.log
2019-10-21T14:24:11.611645Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.18) initializing of server in progress as process 25218
2019-10-21T14:24:12.833917Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: f%t)U*qwM5Pe
2019-10-21T14:24:15.355245Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.18) starting as process 25267
2019-10-21T14:24:15.754979Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2019-10-21T14:24:15.777788Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.18'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  MySQL Community Server - GPL.
2019-10-21T14:24:15.904547Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/run/mysqld/mysqlx.sock' bind-address: '::' port: 33060
```
这里log文件开头第二行就是临时密码，或者直接使用cat /var/log/mysqld.log | grep root@localhost命令查看，我这里的临时密码为：f%t)U*qwM5Pe，我们继续使用临时密码来进入mysql命令行并配置自己的root密码：
```
[root@localhost mysql]# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.18

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
可以看到，使用临时密码已经进入mysql命令行了，接下来进行自己的root密码配置：
```
mysql> alter user "root"@"localhost" identified by "Anshan_pl512";
Query OK, 0 rows affected (0.01 sec)
```
这里需要注意一点：在Mysql 8版本，root密码需要包括大小写字母，特殊字符和数字，我将自己的root密码配置为：Anshan_pl512，注意结尾的;分号。

# 查看所有用户
## 连接mysql服务器
`mysql -u root -p`

## 显示所有database
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| zabbix             |
+--------------------+
5 rows in set (0.01 sec)
```
## 进入“Mysql”库
```
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

## 显示“Mysql”库中的所有表
```
mysql> show tables;
+---------------------------+
| Tables_in_mysql           |
+---------------------------+
| columns_priv              |
| component                 |
| db                        |
| default_roles             |
| engine_cost               |
| func                      |
| general_log               |
| global_grants             |
| gtid_executed             |
| help_category             |
| help_keyword              |
| help_relation             |
| help_topic                |
| innodb_index_stats        |
| innodb_table_stats        |
| password_history          |
| plugin                    |
| procs_priv                |
| proxies_priv              |
| role_edges                |
| server_cost               |
| servers                   |
| slave_master_info         |
| slave_relay_log_info      |
| slave_worker_info         |
| slow_log                  |
| tables_priv               |
| time_zone                 |
| time_zone_leap_second     |
| time_zone_name            |
| time_zone_transition      |
| time_zone_transition_type |
| user                      |
+---------------------------+
33 rows in set (0.01 sec)
```
看到最后一行有个user用户表。
## 显示user表结构
```
mysql> desc user;
+--------------------------+-----------------------------------+------+-----+-----------------------+-------+
| Field                    | Type                              | Null | Key | Default               | Extra |
+--------------------------+-----------------------------------+------+-----+-----------------------+-------+
| Host                     | char(255)                         | NO   | PRI |                       |       |
| User                     | char(32)                          | NO   | PRI |                       |       |
| Select_priv              | enum('N','Y')                     | NO   |     | N                     |       |
| Insert_priv              | enum('N','Y')                     | NO   |     | N                     |       |
| Update_priv              | enum('N','Y')                     | NO   |     | N                     |       |
| Delete_priv              | enum('N','Y')                     | NO   |     | N                     |       |
| Create_priv              | enum('N','Y')                     | NO   |     | N                     |       |
| Drop_priv                | enum('N','Y')                     | NO   |     | N                     |       |
| Reload_priv              | enum('N','Y')                     | NO   |     | N                     |       |
| Shutdown_priv            | enum('N','Y')                     | NO   |     | N                     |       |
| Process_priv             | enum('N','Y')                     | NO   |     | N                     |       |
| File_priv                | enum('N','Y')                     | NO   |     | N                     |       |
| Grant_priv               | enum('N','Y')                     | NO   |     | N                     |       |
| References_priv          | enum('N','Y')                     | NO   |     | N                     |       |
| Index_priv               | enum('N','Y')                     | NO   |     | N                     |       |
| Alter_priv               | enum('N','Y')                     | NO   |     | N                     |       |
| Show_db_priv             | enum('N','Y')                     | NO   |     | N                     |       |
| Super_priv               | enum('N','Y')                     | NO   |     | N                     |       |
| Create_tmp_table_priv    | enum('N','Y')                     | NO   |     | N                     |       |
| Lock_tables_priv         | enum('N','Y')                     | NO   |     | N                     |       |
| Execute_priv             | enum('N','Y')                     | NO   |     | N                     |       |
| Repl_slave_priv          | enum('N','Y')                     | NO   |     | N                     |       |
| Repl_client_priv         | enum('N','Y')                     | NO   |     | N                     |       |
| Create_view_priv         | enum('N','Y')                     | NO   |     | N                     |       |
| Show_view_priv           | enum('N','Y')                     | NO   |     | N                     |       |
| Create_routine_priv      | enum('N','Y')                     | NO   |     | N                     |       |
| Alter_routine_priv       | enum('N','Y')                     | NO   |     | N                     |       |
| Create_user_priv         | enum('N','Y')                     | NO   |     | N                     |       |
| Event_priv               | enum('N','Y')                     | NO   |     | N                     |       |
| Trigger_priv             | enum('N','Y')                     | NO   |     | N                     |       |
| Create_tablespace_priv   | enum('N','Y')                     | NO   |     | N                     |       |
| ssl_type                 | enum('','ANY','X509','SPECIFIED') | NO   |     |                       |       |
| ssl_cipher               | blob                              | NO   |     | NULL                  |       |
| x509_issuer              | blob                              | NO   |     | NULL                  |       |
| x509_subject             | blob                              | NO   |     | NULL                  |       |
| max_questions            | int(11) unsigned                  | NO   |     | 0                     |       |
| max_updates              | int(11) unsigned                  | NO   |     | 0                     |       |
| max_connections          | int(11) unsigned                  | NO   |     | 0                     |       |
| max_user_connections     | int(11) unsigned                  | NO   |     | 0                     |       |
| plugin                   | char(64)                          | NO   |     | caching_sha2_password |       |
| authentication_string    | text                              | YES  |     | NULL                  |       |
| password_expired         | enum('N','Y')                     | NO   |     | N                     |       |
| password_last_changed    | timestamp                         | YES  |     | NULL                  |       |
| password_lifetime        | smallint(5) unsigned              | YES  |     | NULL                  |       |
| account_locked           | enum('N','Y')                     | NO   |     | N                     |       |
| Create_role_priv         | enum('N','Y')                     | NO   |     | N                     |       |
| Drop_role_priv           | enum('N','Y')                     | NO   |     | N                     |       |
| Password_reuse_history   | smallint(5) unsigned              | YES  |     | NULL                  |       |
| Password_reuse_time      | smallint(5) unsigned              | YES  |     | NULL                  |       |
| Password_require_current | enum('N','Y')                     | YES  |     | NULL                  |       |
| User_attributes          | json                              | YES  |     | NULL                  |       |
+--------------------------+-----------------------------------+------+-----+-----------------------+-------+
51 rows in set (0.01 sec)
```
可以看到该表中存在Host和User字段，但没有Password字段，在Mysql现行版本中密码字段不是Password，而是authentication_string。

## 查询所有用户
```
mysql> select User,Host from user;
+------------------+-----------+
| User             | Host      |
+------------------+-----------+
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
| zabbix           | localhost |
+------------------+-----------+
```
这里也可以使用select User,Host,authentication_string from user;带上密码字段查询，但mysql的密码是使用MD5加密的，查看了也没有意义。
