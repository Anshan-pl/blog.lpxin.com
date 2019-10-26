---
title: Zabbix配置
date: 2019-10-25 17:05:19
tags: [zabbix,redhat,rhel,linux,mysql,php,apache]
categories: Zabbix
---

<center>
zabbix可分布式监控
且可以监控服务器、网络设备等
可通过Agent、SNMP、JMX、IPMI进行监控
</center>
<!--more-->

# Zabbix安装
Zabbix的优点，原理等各种文章资料很多，这里直接上使用配置记录，先进行安装。
在之前3.0版本时大部分的运维人员都会通过源码编译安装而不是二进制包进行安装，目前Zabbix官方对于二进制包支持已比较完美，所以这里我们直接使用官方的安装流程进行，[先放上链接，在本文章记录时，Zabbix版本为4.4](https://www.zabbix.com/cn/download?zabbix=4.4&os_distribution=red_hat_enterprise_linux&os_version=7&db=mysql)
所以下面的安装流程直接照搬官方，但毕竟官方的安装流程只是一个参考性，在实际环境中会有些差别，所以这里填上安装中遇到的问题和解决方法。
## 安装Zabbix官方yum源
```
[root@localhost ~]# rpm -Uvh https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/zabbix-release-4.4-1.el7.noarch.rpm
Retrieving https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/zabbix-release-4.4-1.el7.noarch.rpm
warning: /var/tmp/rpm-tmp.jMtluc: Header V4 RSA/SHA512 Signature, key ID a14fe591: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:zabbix-release-4.4-1.el7         ################################# [100%]

[root@localhost ~]# yum clean all
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
Cleaning repos: base extras updates zabbix zabbix-non-supported
Cleaning up list of fastest mirrors
Other repos take up 53 M of disk space (use --verbose for details)
```

## 安装Zabbix server，web UI，agent
这里因为我们是服务端，所以需要安装全部，如果是客户端或者说是被监控端，那么只需要安装一个agent即可：
这里遇到了二进制包下载不下来的问题，修改zabbix yum源中的baseurl http为https。
```
[root@localhost `]# sed -i 's/http:\//https:\//g' /etc/yum.repos.d/zabbix.repo

[root@localhost `]# yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-agent
#回显太长了，这里不放出了
```

## 创建初始数据库
这里需要系统有mysql，[安装mysql 8可以参考之前的文章](https://blog.lpxin.com/2019/10/21/rhel7%E4%B8%8A%E9%85%8D%E7%BD%AEMysql-8/)
```
[root@localhost mysql]# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.0.18 MySQL Community Server - GPL

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database zabbix character set utf8 collate utf8_bin;
Query OK, 1 row affected, 2 warnings (0.02 sec)

mysql> show databases
    -> ;
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

mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'Anshan_pl123';
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'identified by 'Anshan_pl123'' at line 1
```
**这里给zabbix用户授权操作请谨慎操作！为避免不必要的安全问题，请勿设置将所有库权限赋给zabbix**

我这里安装的是mysql 8，而8版本无法给用户授权，意思是不能用grant创建用户，mysql8.0以前的版本可以使用grant在授权的时候隐式的创建用户，8.0以后已经不支持，所以必须先创建用户，然后再授权，命令如下：
```
mysql> create user "zabbix"@"localhost" identified by "Anshan_pl123";
Query OK, 0 rows affected (0.02 sec)

mysql> grant all privileges on zabbix.* to zabbix@localhost;
Query OK, 0 rows affected (0.01 sec)

mysql> quit
Bye
```

## 导入初始架构和数据
```
[root@localhost ~]# zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
Enter password:
```
输入刚刚创建的zabbix用户密码即可，zcat命令表示不真正解压缩文件，只显示压缩包中文件的内容。

## 为Zabbix server配置数据库
编辑配置文件 /etc/zabbix/zabbix_server.conf
```
[root@localhost ~]# sed -i 's/# DBPassword=/DBPassword=Anshan_pl123/g' /etc/zabbix/zabbix_server.conf
```
这里请将Anshan_pl123改为你自己创建的密码。

## 为Zabbix前端配置PHP
编辑配置文件 /etc/httpd/conf.d/zabbix.conf,将php_value date.timezone值修改为你所在的时区：
```
[root@localhost ~]# sed -i 's/# php_value date.timezone Europe\/Riga/php_value date.timezone Asia\/Shanghai/g' /etc/http
d/conf.d/zabbix.conf
```

## 启动Zabbix server和agent进程
如果不想监控自己，那么可以不运行agent，我这里为了演示zabbix使用，所以启动了agent进程：
```
[root@localhost ~]# systemctl restart zabbix-server zabbix-agent httpd

[root@localhost ~]# systemctl enable zabbix-server zabbix-agent httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/zabbix-server.service to /usr/lib/systemd/system/zabbix-server.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/zabbix-agent.service to /usr/lib/systemd/system/zabbix-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
```
到这里Zabbix server端安装以及算是完成了。

## 配置Zabbix前端
访问 http://server_ip_or_name/zabbix 来对zabbix server进行初始化操作并配置。

