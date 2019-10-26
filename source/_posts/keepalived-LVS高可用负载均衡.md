---
title: keepalived+LVS高可用负载均衡
date: 2019-10-22 10:50:56
tags: [keepalived,LVS,高可用,负载均衡,Linux]
categories: Keepalived
---

<center>
利用keepalived开源软件实现高可用的功能
而LVS同样是开源软件实现负载均衡
这篇文章记录利用lvs结合keepalived实现一个高可用的Linux群集系统
</center>
<!--more-->
# keepalived演示
## keepalived安装
一般keepalived在官方源或者光盘源中均有安装包，所以这里直接使用yum安装：
```
[root@localhost ~]# yum install keepalived -y
Failed to set locale, defaulting to C
Loaded plugins: product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
file:///mnt/cdrom/repodata/repomd.xml: [Errno 14] curl#37 - "Couldn't open file /mnt/cdrom/repodata/repomd.xml"
Trying other mirror.
Resolving Dependencies
--> Running transaction check
---> Package keepalived.x86_64 0:1.3.5-16.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

========================================================================================================================
 Package                       Arch                      Version                          Repository               Size
========================================================================================================================
Installing:
 keepalived                    x86_64                    1.3.5-16.el7                     base                    331 k

Transaction Summary
========================================================================================================================
Install  1 Package

Total download size: 331 k
Installed size: 1.0 M
Downloading packages:
keepalived-1.3.5-16.el7.x86_64.rpm                                                               | 331 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : keepalived-1.3.5-16.el7.x86_64                                                                       1/1
  Verifying  : keepalived-1.3.5-16.el7.x86_64                                                                       1/1

Installed:
  keepalived.x86_64 0:1.3.5-16.el7

Complete!
```
## 配置keepalived
安装完成对其后进行配置，这里我使用一主一备模型也就是抢占式配置进行测试，首先对keepalived MASTER进行配置，配置文件默认在/etc/keepalived路径下：
```
[root@localhost ~]# cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
#对默认配置文件进行备份

[root@localhost ~]# \rm /etc/keepalived/keepalived.conf
#删除默认的配置文件

[root@localhost ~]# vim /etc/keepalived/keepalived.conf
#写入以下内容

! Configuration File for keepalived

global_defs {                             #全局配置
  script_user root                        #监控脚本以root身份运行
  enable_script_security                  #如过路径为非root可写，不要配置脚本为root用户执行，简单来说是一个安全机制
}

vrrp_script chk_http                      #定义一个监控脚本，VRRP脚本声明
{
  script "/etc/keepalived/ck_httpd.sh"    #指定脚本路径，注意，请使用绝对路径
  interval 1                              #运行脚本的间隔时间 单位：秒
  weight 2                                #权重，priority值减去此值要小于备服务的priority值
  fall 3                                  #检测几次失败才为失败，整数
  rise 2                                  #检测几次状态为正常的，才确认正常，整数
  user root                               #执行脚本的用户或组，在全局配置中已经定义了运行脚本的用户，所以这里可以不用再次定义
}

vrrp_instance VI_1 {                      #vrrp 实例部分定义，VI_1自定义名称
    state MASTER                          #指定 keepalived 的角色，必须大写 可选值：MASTER|BACKUP
    interface ens33                       #网卡设置，lvs需要绑定在网卡上，realserver绑定在回环口。区别：lvs对访问为外，realserver为内不易暴露本机信息
    !mcast_src_ip 192.168.122.34
    virtual_router_id 51                  #虚拟路由标识，是一个数字，同一个vrrp 实例使用唯一的标识，MASTER和BACKUP 的 同一个 vrrp_instance 下 这个标识必须保持一致
    priority 100                          #定义优先级，数字越大，优先级越高，请注意，同一组高可用节点中，谁的优先级最大谁就是master节点，而不是对比state的值，只是state不同，其weight不同。
    advert_int 1                          #设定 MASTER 与 BACKUP 负载均衡之间同步检查的时间间隔，单位为秒，两个节点设置必须一样
    authentication {                      #设置验证类型和密码，同一个高可用组内节点必须一致
        auth_type PASS
        auth_pass 1111
    }
    track_script {                        #脚本监控状态
        chk_http                          #指定定义的脚本
        !chk_http weight 5                #可加权重，但会覆盖声明的脚本权重值
    }
    virtual_ipaddress {                   #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
        192.168.199.10/24
    }
    notify_master "/etc/keepalived/start_haproxy.sh master"    #当前节点成为master时，通知脚本执行任务
    notify_backup "/etc/keepalived/start_haproxy.sh backup"    #当前节点成为backup时，通知脚本执行任务
    notify_fault  "/etc/keepalived/start_haproxy.sh stop"      #当当前节点出现故障，执行的任务 
}
```
这里需要注意下，keepalived的配置注释符是！叹号而不是#井字符，所以如果你使用我上面给出的例子配置文件，那么请删除所有的#注释说明。
在global_defs中可以定义报警通知的邮箱，我这里没有配置，百度上很多。因为我的内网段为192.168.199，所以这里我配置的VIP为192.168.199.10。

## 监控脚本ck_http
并且这里我用到了两个脚本，分别为chk_http，http服务的监控脚本和tart_haproxy节点状态监控脚本。
先给出chk_http监控脚本：
```
#!/bin/bash

count=$(ps -C httpd --no-heading | wc -l)
if [ "$count" = "0" ];then
  systemctl restart httpd
  sleep 3
  count=$(ps -C httpd --no-heading | wc -l)
  if [ "$count" = "0" ];then
    systemctl stop keepalived
  fi
fi

exit 0

[root@localhost ~]# chmod 755 /etc/keepalived/ck_httpd.sh
```
该脚本使用ps -C命令每1秒运行一次检查httpd进程是否存活，不存活那么就重启httpd服务并再次进行检查，如果还是不存活，那么就停止该节点的keepalived服务，启用backup节点，最后不要忘记给脚本赋权。
继续给出start_haproxy脚本：
```
#!/bin/bash
#This is test

time=`date +"%Y-%m-%d %H:%M:%S"`
master()
{
    echo ${time}": Now, the node is master" >> /root/haproxy.log
}

backup()
{
    echo ${time}": Now, the node is backup" >> /root/haproxy.log
}

stop()
{
    echo ${time}": Node stop" >> /root/haproxy.log
}

case $1 in
    master )
        master
        ;;
    backup )
        backup
        ;;
    stop )
        stop
        ;;
esac

[root@localhost ~]# chmod 755 /etc/keepalived/start_haproxy.sh
```
该脚本只是单纯用来演示notify_master、notify_backup、notify_fault三参数段是否可用，脚本非常简单所以不多做解释。

## httpd服务
为了演示keepalived高可用，这里我们直接使用http来提供服务：
```
[root@localhost ~]# yum install httpd
Failed to set locale, defaulting to C
Loaded plugins: product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
file:///mnt/cdrom/repodata/repomd.xml: [Errno 14] curl#37 - "Couldn't open file /mnt/cdrom/repodata/repomd.xml"
Trying other mirror.
Resolving Dependencies
--> Running transaction check
---> Package httpd.x86_64 0:2.4.6-90.el7.centos will be installed
--> Processing Dependency: httpd-tools = 2.4.6-90.el7.centos for package: httpd-2.4.6-90.el7.centos.x86_64
--> Running transaction check
---> Package httpd-tools.x86_64 0:2.4.6-90.el7 will be updated
---> Package httpd-tools.x86_64 0:2.4.6-90.el7.centos will be an update
--> Finished Dependency Resolution

Dependencies Resolved

========================================================================================================================
 Package                      Arch                    Version                               Repository             Size
========================================================================================================================
Installing:
 httpd                        x86_64                  2.4.6-90.el7.centos                   base                  2.7 M
Updating for dependencies:
 httpd-tools                  x86_64                  2.4.6-90.el7.centos                   base                   91 k

Transaction Summary
========================================================================================================================
Install  1 Package
Upgrade             ( 1 Dependent package)

Total download size: 2.8 M
Is this ok [y/d/N]: y
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
(1/2): httpd-tools-2.4.6-90.el7.centos.x86_64.rpm                                                |  91 kB  00:00:00
(2/2): httpd-2.4.6-90.el7.centos.x86_64.rpm                                                      | 2.7 MB  00:00:01
------------------------------------------------------------------------------------------------------------------------
Total                                                                                   2.4 MB/s | 2.8 MB  00:00:01
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : httpd-tools-2.4.6-90.el7.centos.x86_64                                                               1/3
  Installing : httpd-2.4.6-90.el7.centos.x86_64                                                                     2/3
  Cleanup    : httpd-tools-2.4.6-90.el7.x86_64                                                                      3/3
  Verifying  : httpd-tools-2.4.6-90.el7.centos.x86_64                                                               1/3
  Verifying  : httpd-2.4.6-90.el7.centos.x86_64                                                                     2/3
  Verifying  : httpd-tools-2.4.6-90.el7.x86_64                                                                      3/3

Installed:
  httpd.x86_64 0:2.4.6-90.el7.centos

Dependency Updated:
  httpd-tools.x86_64 0:2.4.6-90.el7.centos

Complete!

[root@localhost ~]# echo "This is node1" > /var/www/html/index.html

[root@localhost ~]# systemctl stop firewalld

[root@localhost ~]# systemctl start httpd

[root@localhost ~]# curl 127.0.0.1
This is node1
```
httpd服务部署完成，下面开始搭建backup节点环境，全部过程和master节点相同，只是keepalived.conf配置文件中的
```
state MASTER    -->    state BACKUP
priority 100    -->    priority 50
```
两个节点都部署完成后，启动master和backup节点，查看日志并尝试从VIP访问：
```
[root@localhost ~]# setenforce 0
#SELinux一定要关闭，否则将影响监控脚本的运行

[root@localhost ~]# systemctl start keepalived

[root@localhost ~]# cat /var/log/messages
-------------------------省略部分输出-------------------------
Oct 22 23:29:25 localhost Keepalived[5815]: Starting Keepalived v1.3.5 (03/19,2017), git commit v1.3.5-6-g6fa32f2
Oct 22 23:29:25 localhost Keepalived[5815]: Opening file '/etc/keepalived/keepalived.conf'.
Oct 22 23:29:25 localhost Keepalived[5816]: Starting Healthcheck child process, pid=5817
Oct 22 23:29:25 localhost systemd: Can't open PID file /var/run/keepalived.pid (yet?) after start: No such file or directory
Oct 22 23:29:25 localhost Keepalived[5816]: Starting VRRP child process, pid=5818
Oct 22 23:29:25 localhost systemd: Started LVS and VRRP High Availability Monitor.
Oct 22 23:29:25 localhost Keepalived_healthcheckers[5817]: Opening file '/etc/keepalived/keepalived.conf'.
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: Registering Kernel netlink reflector
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: Registering Kernel netlink command channel
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: Registering gratuitous ARP shared channel
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: Opening file '/etc/keepalived/keepalived.conf'.
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) removing protocol VIPs.
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: Using LinkWatch kernel netlink reflector...
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: VRRP sockpool: [ifindex(2), proto(112), unicast(0), fd(10,11)]
Oct 22 23:29:25 localhost Keepalived_vrrp[5818]: VRRP_Script(chk_http) succeeded
Oct 22 23:29:26 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) Transition to MASTER STATE
Oct 22 23:29:26 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) Changing effective priority from 50 to 52
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) Entering MASTER STATE
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) setting protocol VIPs.
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on ens33 for 192.168.199.10
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:27 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:32 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:32 localhost Keepalived_vrrp[5818]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on ens33 for 192.168.199.10
Oct 22 23:29:32 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:32 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:32 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 22 23:29:32 localhost Keepalived_vrrp[5818]: Sending gratuitous ARP on ens33 for 192.168.199.10

[root@localhost ~]# cat /root/haproxy.log
2019-10-22 23:29:27: Now, the node is master
```
基本上这里/root/haproxy.log文件已经生成，并且在messages中看到VRRP_Script(chk_http) succeeded就没问题了，可以手动停止httpd服务，keepalived监控脚本会自动拉起：
```
[root@localhost ~]# systemctl stop httpd

[root@localhost ~]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-10-22 23:23:36 CST; 34s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 4087 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├4087 /usr/sbin/httpd -DFOREGROUND
           ├4088 /usr/sbin/httpd -DFOREGROUND
           ├4089 /usr/sbin/httpd -DFOREGROUND
           ├4090 /usr/sbin/httpd -DFOREGROUND
           ├4091 /usr/sbin/httpd -DFOREGROUND
           └4092 /usr/sbin/httpd -DFOREGROUND

Oct 22 23:23:36 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
Oct 22 23:23:36 localhost.localdomain httpd[4087]: AH00558: httpd: Could not reliably determine the server's full...sage
Oct 22 23:23:36 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.
```
backup上同样启动keepalived，但是这里backup节点是不会运行监控脚本的，只有当它成为master节点后，才会持续运行监控。
在master节点上停止keepalived后，观察backup节点：
```
[root@localhost ~]# tail -f /var/log/messages

-------------------------省略部分输出-------------------------
Oct 23 00:25:58 localhost Keepalived_vrrp[19797]: VRRP_Instance(VI_1) Transition to MASTER STATE
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: VRRP_Instance(VI_1) Entering MASTER STATE
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: VRRP_Instance(VI_1) setting protocol VIPs.
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on ens33 for 192.168.199.10
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:25:59 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:26:04 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:26:04 localhost Keepalived_vrrp[19797]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on ens33 for 192.168.199.10
Oct 23 00:26:04 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:26:04 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:26:04 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
Oct 23 00:26:04 localhost Keepalived_vrrp[19797]: Sending gratuitous ARP on ens33 for 192.168.199.10
```
可以看到，我在master节点上停止keepalived服务后backup节点自动升级为master节点并获取VIP对外提供服务。
这里我没有进行web访问测试，只要网络畅通，且messages没有error，防火墙关闭，那么基本没有问题。
提醒两点：
- 我在使用keeplived时遇到了一个bug，不要直接使用命令systemctl restart keepalived重启服务，必须要先停止，再启动，否则有Can't open pid的问题。
- 测试时请一定要关闭SELinux，否则将影响监控脚本的运行。

# LVS演示

## LVS原理
LVS由前端的负载均衡器(Load Balancer，LB)和后端的真实服务器(Real Server，RS)群组成。RS间可通过局域网或广域网连接。LVS的这种结构对用户是透明的，用户只能看见一台作为LB的虚拟服务器(Virtual Server)，而看不到提供服务的RS群。当用户的请求发往虚拟服务器，LB根据设定的包转发策略和负载均衡调度算法将用户请求转发给RS。RS再将用户请求结果返回给用户。同请求包一样，应答包的返回方式也与包转发策略有关。

## LVS四种工作模式
- Virtual Server via Network Address Translation（VS/NAT）:
    通过网络地址转换，调度器重写请求报文的目标地址，根据预设的调度算法，将请求分派给后端的真实服务器；真实服务器的响应报文通过调度器时，报文的源地址被重写，再返回给客户，完成整个负载调度过程。

- Virtual Server via IP Tunneling（VS/TUN)
    采用NAT技术时，由于请求和响应报文都必须经过调度器地址重写，当客户请求越来越多时，调度器的处理能力将成为瓶颈。为了解决这个问题，调度器把请求报 文通过IP隧道转发至真实服务器，而真实服务器将响应直接返回给客户，所以调度器只处理请求报文。由于一般网络服务应答比请求报文大许多，采用 VS/TUN技术后，集群系统的最大吞吐量可以提高10倍。

- Virtual Server via Direct Routing（VS/DR）
    VS/DR通过改写请求报文的MAC地址，将请求发送到真实服务器，而真实服务器将响应直接返回给客户。同VS/TUN技术一样，VS/DR技术可极大地 提高集群系统的伸缩性。这种方法没有IP隧道的开销，对集群中的真实服务器也没有必须支持IP隧道协议的要求，但是要求调度器与真实服务器都有一块网卡连 在同一物理网段上。

- FULLNAT：大局域网间
    该模型仅在一些文档中提及。

这里只介绍三种最常用的模式：NAT、TUN和DR模式

## NAT模式
实验环境：
DS：redhat 7
ens33：192.168.199.205（内网）
ens38：192.168.2.133（外网）

RS1：redhat 7
ens33： 192.168.199.233

RS2：redhat7
ens33：192.168.199.229

client：windows10
VMnet1：192.168.2.1

### LVS调度器（DS）
安装ipvsadm管理工具
```
[root@localhost ~]# yum install ipvsadm -y
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
Determining fastest mirrors
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
base                                                                                             | 3.6 kB  00:00:00
extras                                                                                           | 2.9 kB  00:00:00
updates                                                                                          | 2.9 kB  00:00:00
(1/4): base/x86_64/group_gz                                                                      | 165 kB  00:00:00
(2/4): extras/x86_64/primary_db                                                                  | 153 kB  00:00:00
(3/4): updates/x86_64/primary_db                                                                 | 2.8 MB  00:00:01
base/x86_64/primary_db         FAILED
http://mirrors.aliyuncs.com/centos/7/os/x86_64/repodata/04efe80d41ea3d94d36294f7107709d1c8f70db11e152d6ef562da344748581a-primary.sqlite.bz2: [Errno 12] Timeout on http://mirrors.aliyuncs.com/centos/7/os/x86_64/repodata/04efe80d41ea3d94d36294f7107709d1c8f70db11e152d6ef562da344748581a-primary.sqlite.bz2: (28, 'Connection timed out after 30002 milliseconds')
Trying other mirror.
(4/4): base/x86_64/primary_db                                                                    | 6.0 MB  00:00:02
Resolving Dependencies
--> Running transaction check
---> Package ipvsadm.x86_64 0:1.27-7.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

========================================================================================================================
 Package                     Arch                       Version                          Repository                Size
========================================================================================================================
Installing:
 ipvsadm                     x86_64                     1.27-7.el7                       base                      45 k

Transaction Summary
========================================================================================================================
Install  1 Package

Total download size: 45 k
Installed size: 75 k
Downloading packages:
ipvsadm-1.27-7.el7.x86_64.rpm                                                                    |  45 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ipvsadm-1.27-7.el7.x86_64                                                                            1/1
  Verifying  : ipvsadm-1.27-7.el7.x86_64                                                                            1/1

Installed:
  ipvsadm.x86_64 0:1.27-7.el7

Complete!
```
目前主流的操作系统内核中都已经集成了ip_vs模块，使用lsmod查看：
```
[root@localhost ~]# lsmod | grep ip_vs
ip_vs_rr               12600  1
ip_vs                 145497  3 ip_vs_rr
nf_conntrack          139224  1 ip_vs
libcrc32c              12644  3 xfs,ip_vs,nf_conntrack
```
可以看到，我这里的测试环境系统已经集成了ip_vs模块。
接下来需要配置ip转发，因为我们需要让通过DS的请求包转发到真正提供服务的RS上进行处理：
```
[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/ip_forward
#该命令是临时打开ipv4的转发功能

[root@localhost ~]# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf

[root@localhost ~]# sysctl -p
net.ipv4.ip_forward = 1
#将ipv4转发写入到配置文件中，使其永久生效
```
关闭防火墙和SELinux避免对实验造成影响：
```
[root@localhost ~]# setenforce 0

[root@localhost ~]# getenforce
Permissive
#该命令临时关闭SELinux

[root@localhost ~]# vim /etc/sysconfig/selinux

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

#将SELINUX=enforcing项改为permissive，使其永久关闭


[root@localhost ~]# systemctl stop firewalld
#临时firewalld防火墙

[root@localhost ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
#禁止其开启启动服务，使其永久关闭
```
现在可以进行LVS的NAT模式配置了，先查看下双网卡的ip：
```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:df:73:f7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.199.205/24 brd 192.168.199.255 scope global noprefixroute dynamic ens33
       valid_lft 38087sec preferred_lft 38087sec
    inet6 fe80::20c:29ff:fedf:73f7/64 scope link
       valid_lft forever preferred_lft forever
3: ens38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:df:73:01 brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.133/24 brd 192.168.2.255 scope global noprefixroute dynamic ens38
       valid_lft 1603sec preferred_lft 1603sec
    inet6 fe80::c79b:35d:b44f:d9f9/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
配置ipvsadm
ipvsadm参数简单理解：
```
添加虚拟服务器
    语法:ipvsadm -A [-t|u|f]  [vip_addr:port]  [-s:指定算法]
    -A:添加
    -t:TCP协议
    -u:UDP协议
    -f:防火墙标记
    -D:删除虚拟服务器记录
    -E:修改虚拟服务器记录
    -C:清空所有记录
    -L:查看
添加后端RealServer
    语法:ipvsadm -a [-t|u|f] [vip_addr:port] [-r ip_addr] [-g|i|m] [-w 指定权重]
    -a:添加
    -t:TCP协议
    -u:UDP协议
    -f:防火墙标记
    -r:指定后端realserver的IP
    -g:DR模式
    -i:TUN模式
    -m:NAT模式
    -w:指定权重
    -d:删除realserver记录
    -e:修改realserver记录
    -l:查看
通用:
    ipvsadm -Ln:查看当前配置的虚拟服务和各个RS的权重
    ipvsadm --save:保存规则到配置文件
    ipvsadm -lnc:查看当前ipvs模块中记录的连接（可用于观察转发情况）
```
LVS集群的负载调度主要是由工作在内核当中的IPVS IP负载均衡软件负责进行调度的，IPVS在内核中的负载均衡调度是以连接为粒度的,在内核中的连接调度算法上，IPVS已实现了以下八种调度算法：
-  轮询调度（Round-Robin Scheduling）
   这种算法就是以轮叫的方式依次将请求调度不同的服务器，算法的优点是其简洁性，它无需记录当前所有连接的状态，所以它是一种无状态调度。轮叫调度算法假设所有服务器处理性能均相同，不管服务器的当前连接数和响应速度。该算法相对简单，不适用于服务器组中处理性能不一的情况，而且当请求服务时间变化比较大时，轮叫调度算法容易导致服务器间的负载不平衡。

-  加权轮询调度（Weighted Round-Robin Scheduling）
   这种算法可以解决服务器间性能不一的情况，它用相应的权值表示服务器的处理性能，服务器的缺省权值为1。假设服务器A的权值为1，B的 权值为2，则表示服务器B的处理性能是A的两倍。加权轮叫调度算法是按权值的高低和轮叫方式分配请求到各服务器。权值高的服务器先收到的连接，权值高的服 务器比权值低的服务器处理更多的连接，相同权值的服务器处理相同数目的连接数。

-  最小连接调度（Least-Connection Scheduling）
   这种算法是把新的连接请求分配到当前连接数最小的服务器。最小连接调度是一种动态调度算法，它通过服务器当前所活跃的连接数来估计服务 器的负载情况。调度器需要记录各个服务器已建立连接的数目，当一个请求被调度到某台服务器，其连接数加1；当连接中止或超时，其连接数减一。当各个服务器有相同的处理性能时，最小连接调度算法能把负载变化大的请求分布平滑到各个服务器上，所有处理时间比较长的请求不可能被发送到同一台服 务器上。但是，当各个服务器的处理能力不同时，该算法并不理想，因为TCP连接处理请求后会进入TIME_WAIT状态，TCP的TIME_WAIT一般 为2分钟，此时连接还占用服务器的资源，所以会出现这样情形，性能高的服务器已处理所收到的连接，连接处于TIME_WAIT状态，而性能低的服务器已经 忙于处理所收到的连接，还不断地收到新的连接请求。

-  加权最小连接调度（Weighted Least-Connection Scheduling）
   这种算法是最小连接调度的超集，各个服务器用相应的权值表示其处理性能。服务器的缺省权值为1，系统管理员可以动态地设置服务器的权 值。加权最小连接调度在调度新连接时尽可能使服务器的已建立连接数和其权值成比例。

-  基于局部性的最少链接（Locality-Based Least Connections Scheduling）
   这种算法是请求数据包的目标 IP 地址的一种调度算法，该算法先根据请求的目标 IP 地址寻找最近的该目标 IP 地址所有使用的服务器，如果这台服务器依然可用，并且有能力处理该请求，调度器会尽量选择相同的服务器，否则会继续选择其它可行的服务器

-  带复制的基于局部性最少链接（Locality-Based Least Connections with Replication Scheduling）
   这种算法先根据请求的目标IP地址找出该目标IP地址对应的服务器组；按“最小连接”原则从该服务器组中选出一台服务器，若服务器没有超载， 将请求发送到该服务器；若服务器超载；则按“最小连接”原则从整个集群中选出一台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。同时，当该 服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的程度。

-  目标地址散列调度（Destination Hashing Scheduling）
   此算法先根据请求的目标IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

-  源地址散列调度（Source Hashing Scheduling）
   此算法根据请求的源IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。它采用的散列函数与目标地址散列调度算法 的相同。它的算法流程与目标地址散列调度算法的基本相似。

我这里只是测试httpd，不需要多端口，所以这里只绑定一个80端口：
```
[root@localhost ~]# ipvsadm -A -t 192.168.2.133:80 -s rr
#VIP后跟端口号是只绑定单端口

[root@localhost ~]# ipvsadm -A -t 192.168.2.133 -s rr -p 900
#如果不想指定端口，即开发全端口，那么需要加上-p参数，指定timeout。

[root@localhost ~]# ipvsadm -a -t 192.168.2.133:80 -r 192.168.199.233 -m
[root@localhost ~]# ipvsadm -a -t 192.168.2.133:80 -r 192.168.199.229 -m
#配置负载均衡
```
LVS调度器到这里配置结束

### 后端真实服务器（RS）
查看网卡信息：
```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:18:c2:7d brd ff:ff:ff:ff:ff:ff
    inet 192.168.199.233/24 brd 192.168.199.255 scope global noprefixroute dynamic ens33
       valid_lft 32807sec preferred_lft 32807sec
    inet6 fe80::20c:29ff:fe18:c27d/64 scope link
       valid_lft forever preferred_lft forever
```
关闭防火墙，启动httpd服务，这里httpd服务配置请参考1.4
```
[root@localhost ~]# systemctl stop firewalld
[root@localhost ~]# systemctl start httpd
```
重新添加一条默认路由，指向DS调度器，实现路由转发。
这里提醒一下，RS采用双网卡机制很容易出现，TCP连接问题，也就是syn消息收不到ack响应，所以尽量将无关的网卡禁用，以免路由配置的复杂。
```
[root@localhost ~]# route add default gw 192.168.199.205
```
RS上的配置结束，NAT模式就是这么简单，从client上进行访问测试。
root@DESKTOP-GE9K8LB:/mnt/c/Users/Anshan/Desktop# curl 192.168.2.133
This is node2
root@DESKTOP-GE9K8LB:/mnt/c/Users/Anshan/Desktop# curl 192.168.2.133
This is node1

### ipvsadm配置永久保存
```
[root@localhost ~]# ipvsadm --save
-A -t localhost.localdomain:http -s rr
-a -t localhost.localdomain:http -r 192.168.199.229:http -m -w 1
-a -t localhost.localdomain:http -r 192.168.199.233:http -m -w 1
#保存到配置文件

[root@localhost ~]# systemctl enable ipvsadm
Created symlink from /etc/systemd/system/multi-user.target.wants/ipvsadm.service to /usr/lib/systemd/system/ipvsadm.service.
# 添加ipvsadm到开机自启项
```
### 停止LVS
删除路由以及清除ipvsadm配置命令，分别在RS和DS上执行：
```
[root@localhost ~]# route del -net 0.0.0.0 gw 192.168.199.205
#RS

[root@localhost ~]# ipvsadm -C
#DS
```

## TUN模式
实验环境同NAT模式

因为VS/TUN模式下，使用的是IP隧道把数据包从DS分发到各RS节点的。所以需要所有Linux主机（Director和RealServer）都支持一种ipip隧道协议。这就需要在Linux内核中有支持ipip协议（一种IP隧道协议）的模块。使用lsmod命令查看：
```

```


### DS上使用单张内网卡实现TUN
#### 从属ip：
这里是在DS调度器上对内网网卡配置从属IP：
```
[root@localhost ~]# ifconfig ens33:0 192.168.199.10 netmask 255.255.255.0 up
#当然，这里也可以将DS中的VIP配置到虚拟网卡tunl0上

ifconfig tunl0 192.168.199.10 netmask 255.255.255.255 up
```

#### ipvsadm配置
```
[root@localhost ~]# ipvsadm -A -t 192.168.199.10:80 -s rr

[root@localhost ~]# ipvsadm -a -t 192.168.199.10:80 -r 192.168.199.233 -i

[root@localhost ~]# ipvsadm -a -t 192.168.199.10:80 -r 192.168.199.229 -i
#-i参数表示TUN模式
```
防火墙关闭，SELinux关闭，至此单内网卡DS调度器上的配置工作结束。
需要特别说明的是：在VS/TUN模块下Director是不需要开启ip_forword(路由转发)功能的。

#### RS后端服务器配置
对于RS的配置基本同DS，首先需要把VIP配置在RealServer节点的tunl0网卡上：
`[root@localhost ~]# ifconfig tunl0 192.168.199.10 netmask 255.255.255.255 up`

#### 关闭反向路由验证
关闭tunl0网卡的反向路由校验（默认情况下是开着的，基值为1）。因为对tunl0网卡上反向路由的校验策略使用的是all上和tunl0网卡上两rp_filter参数中的较大值。所以需要同时对all中rp_filter参数进行设置。
```
[root@localhost ~]# echo "0" > /proc/sys/net/ipv4/conf/tunl0/rp_filter
[root@localhost ~]# echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
```
因为在VS/TUN模式下，所有RealServer都是可以与client进行直接通信的，所以也可以不关闭反向路由校验，而是把校验规则设置的宽松一些（把rp_filter的值设置为2）。
`echo "2" > /proc/sys/net/ipv4/conf/tunl0/rp_filter`

以上就是所有针对LVS在VS/TUN模式下单内网卡RealServer的配置了，同样，RS需要关闭防火墙，或配置防火墙以允许ipip协议数据包通过，在client上访问VIP检查效果。
### DS双网卡实现TUM
双网卡即使用内网卡与后端RS服务器进行通信，外网卡对外提供服务，相比较单网卡配置，少了从属ip的步骤：

#### DS端ipvsadm配置
```
[root@localhost ~]# ipvsadm -A -t 192.168.2.133:80 -s rr

[root@localhost ~]# ipvsadm -a -t 192.168.2.133:80 -r 192.168.199.233 -i

[root@localhost ~]# ipvsadm -a -t 192.168.2.133:80 -r 192.168.199.229 -i
```

#### RS后端服务器配置
对于后端服务器一样需要tunl0虚拟网卡：
```
[root@localhost ~]# ifconfig tunl0 192.168.2.133 netmask 255.255.255.255 up
```
#### 关闭反向路由验证
这里同单网卡配置，关闭tunl0网卡的反向路由校验：
```
[root@localhost ~]# echo "0" > /proc/sys/net/ipv4/conf/tunl0/rp_filter
[root@localhost ~]# echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
```

#### arp抑制
这里因为DS上使用了双网卡不同网段，但RS上的arp请求可以通过DS访问到VIP，所以需要对RS端进行arp抑制：
```
[root@localhost ~]# echo "1" >/proc/sys/net/ipv4/conf/tunl0/arp_ignore
[root@localhost ~]# echo "2" >/proc/sys/net/ipv4/conf/tunl0/arp_announce
[root@localhost ~]# echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
[root@localhost ~]# echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```
至此，双网卡TUN模式测试环境搭建完成，在client上访问VIP查看效果即可。

## DR模式
实验环境同NAT模式。
### DS上使用单内网卡实现DR
单内网卡实现DR需要先配置一个从属ip（VIP）：
```
[root@localhost ~]# ifconfig ens33:0 192.168.199.10/24
#使用ifconfig和ip命令都可以实现添加ip

[root@localhost ~]# ip addr add 192.168.199.10/24 dev ens33:0
#注意，只是使用ip addr命令添加的从属ip，使用ifconfig是不显示的，可以用ip a查看
```
#### 配置ipvsadm
```
[root@localhost ~]# ipvsadm -A -t 192.168.199.10:80 -s rr
[root@localhost ~]# ipvsadm -a -t 192.168.199.10:80 -r 192.168.199.233 -g
[root@localhost ~]# ipvsadm -a -t 192.168.199.10:80 -r 192.168.199.229 -g
```
关闭防火墙，关闭SELinux，DS调度器上的配置已经完成，继续配置后端服务器RS。
#### RS上给lo回环配置从属ip
因为此时绑定的网络接口不进行对外通信，所以VIP绑定在lo的别名上
```
[root@localhost ~]# ifconfig lo:0 192.168.199.10  netmask 255.255.255.255 broadcast 192.168.199.10
```
#### 抑制arp
RS上关闭arp通告、回应：
```
[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/conf/ens33/arp_ignore
[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
[root@localhost ~]# echo 2 > /proc/sys/net/ipv4/conf/ens33/arp_announce
[root@localhost ~]# echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```
关闭防火墙，启动httpd服务，RS上的配置也到此结束，使用client访问VIP查看效果。

### DS上使用双网卡实现DR
基本配置同单内网卡，只是在DS调度器上不需要再添加从属ip，将VIP改为外网IP即可，不再做演示。

## LVS的VIP端口，使用netstat无法查看
使用LVS监听端口，使用netstat命令是无法查找到对应端口的开放状态的，那么这里就存在一个问题，LVS和web服务都可以正常监听80端口，岂不是会造成端口冲突？
答案是不会的，因为LVS规则算是内核方法，使用netstat -ntulp显示不了其侦听的端口。如果有用户空间同端口开放，那么当客户端访问VIP时，按LVS的配置将其转发给后端服务器，如果访问的不是VIP，那么将使用本机的服务提供响应！

# keepalived+LVS实现高可用负载均衡
因为该教程已经演示过keepalive高可用和LVS三种常用负载均衡模式配置方法，所以这里直接演示keepalived+LVS的DR模式实现高可用负载均衡。

这里keepalived+LVS实验环境由2台DS调度器，2台RS后端服务器如下：

DS1：redhat 7
ens33：192.168.199.205（内网）
ens38：192.168.2.133（外网）

DS2：redhat 7
ens33：192.168.199.223（内网）
ens38：192.168.2.134（外网）

RS1：redhat 7
ens33： 192.168.199.229

RS2：redhat7
ens33：192.168.199.190

client：windows10
VMnet1：192.168.2.1
## 配置keepalived
```
! Configuration File for keepalived

global_defs {
    !router_id LVS_DEVEL                     #运行keepalived机器的一个标识，可忽略
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.199.10
    }
}

#添加虚拟服务器（LVS）
#相当于 ipvsadm -A -t 192.168.199.10:80 -s rr
virtual_server 192.168.199.10 80 {
    delay_loop 6                            #服务健康检查周期,单位是秒
    lb_algo rr                              #调度算法
    lb_kind DR                              #设置LVS负载均衡模式为dr机制(NAT、TUN、DR)
    persistence_timeout 50                  #会话保持时间，单位是秒
    protocol TCP                            #指定转发协议类型(TCP、UDP)

    sorry_server 127.0.0.1 80               #当所有的real_server都不可用时将请求转发到该server上

    #添加后端realserver
    #相当于 ipvsadm -a -t 192.168.199.10:80 -r 192.168.199.229:80 -w 1
    real_server 192.168.199.229 80  {
        weight 1                            #配置节点权值，数字越大权重越高，对应ipvsadm的-w参数
        TCP_CHECK {                         #健康检查方式
            connect_timeout 8               #超时时间
            nb_get_retry 3                  #重试次数
            delay_before_retry 3            #重试间隔
            connect_port 80                 #检查时连接的端口
        }
    }

   #相当于 ipvsadm -a -t 192.168.199.10:80 -r 192.168.199.190:80 -w 1
   real_server 192.168.199.190 80  {
        weight 1
        TCP_CHECK {
            connect_timeout 8
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
}

```

DS1上master节点keepalived配置结束，将该配置拷贝到DS2上将：
MASTER改为BACKUP
priority 100参数值改为50即可。

## 配置realserver
配置VIP到本地回环网卡lo上，并只广播自己
`ifconfig lo:0 192.168.199.10 broadcast 192.168.199.10 netmask 255.255.255.255 up`
抑制arp
```
[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/conf/ens33/arp_ignore
[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
[root@localhost ~]# echo 2 > /proc/sys/net/ipv4/conf/ens33/arp_announce
[root@localhost ~]# echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```
在两个DS服务器上均进行上例操作并关闭防火墙，启动httpd服务即可，使用Client访问VIP查看效果。
到这里keepalived+LVS高可用负载均衡全部配置完毕。
