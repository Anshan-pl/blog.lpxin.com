---
title: CentOS 7双网卡配置dhcpd服务器实现局域网访问外网
date: 2019-05-15 16:17:29
tags: [CentOS,dhcp]
categories: CentOS
---
<center>
如果你需要在双网卡的Linux系统上同时实现拨号和dhcpd服务（单臂路由不在讨论范围内）<br/>
请继续阅读↓
</center>
<!--more-->

## DHCP具有以下功能：
DHCP协议采用客户端/服务器模型，主机地址的动态分配任务由网络主机驱动。当DHCP服务器接收到来自网络主机申请地址的信息时，才会向网络主机发送相关的地址配置等信息，以实现网络主机地址信息的动态配置.
### 1. 保证任何IP地址在同一时刻只能由一台DHCP客户机所使用。
### 2. DHCP应当可以给用户分配永久固定的IP地址。
### 3. DHCP应当可以同用其他方法获得IP地址的主机共存（如手工配置IP地址的主机）。
### 4. DHCP服务器应当向现有的BOOTP客户端提供服务。

简单来说，DHCP就是我们平常使用路由器通过LAN口或者WIFI连接时，路由器给我们自动分配的局域网IP地址，这里的路由器就是我们的上级网关。

## 网桥配置
### 1、CentOS 7默认带有在系统启动时加载的桥接模块。使用以下命令验证模块是否已加载：
`modinfo bridge`
### 2、因为我这里是之前记录的doc文档，所以没有图或者输出信息保存了，只要你运行上面这条命令之后有回显就表示模块已加载。
如果未加载模块，则可以使用以下命令加载它：
`modprobe --first-time bridge`
### 3、安装bridge-utils以控制网络适配器：
`yum install bridge-utils vim -y`
### 4、安装完成后，我们创建属于自己的网桥br0：
`brctl addbr br0`
### 5、配置ifcfg-ens33，此网卡为通外网的网卡（可以理解为路由器的WAN口）：
```
vim /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE=Ethernet
BOOTPROTO="static"
IPADDR="192.168.0.3"
NETMASK="255.255.255.0"
GATEWAY="192.168.0.1"
DNS1="192.168.0.1"
NM_CONTROLLED="no"
#BRIDGE="br0"
DEVICE="ens33"
ONBOOT="yes"
#PROMISC=yes

#注意，这里网口名可能不一样，看自己的机器而定！
```
### 6、配置ifcfg-ens37，此网卡为内网网关，并绑定到网桥br0：
```
vim /etc/sysconfig/network-scripts/ifcfg-ens37
TYPE=Ethernet
BOOTPROTO="static"
IPADDR="192.168.10.1"
NETMASK="255.255.255.0"
DNS1=119.29.29.29
NM_CONTROLLED="no"
DEVICE="ens37"
ONBOOT="yes"
BRIDGE="br0"

```
### 7、配置网桥br0：
```
vim /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE="br0"
TYPE="Bridge"
ONBOOT="yes"
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.10.1
NETMASK=255.255.255.0
GATEWAY=
DNS1=119.29.29.29
#DEFROUTE=yes
#IPV6INIT=no
#USERCTL=no

```
### 8、重启网络：
`systemctl restart network`
### 9、查看网络可以看到br0，ip地址也都正确之后就可以进行dhcpd配置了。
```
ifconfig
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.10.1  netmask 255.255.255.0  broadcast 192.168.10.255
        inet6 fe80::20c:29ff:fe5c:3120  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:5c:31:2a  txqueuelen 1000  (Ethernet)
        RX packets 9817  bytes 537701 (525.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2490  bytes 268317 (262.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.3  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:fe5c:3120  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:5c:31:20  txqueuelen 1000  (Ethernet)
        RX packets 4868408  bytes 7256934168 (6.7 GiB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 183643  bytes 12908703 (12.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens37: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::20c:29ff:fe5c:312a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:5c:31:2a  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5993  bytes 377406 (368.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 95  bytes 16296 (15.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 95  bytes 16296 (15.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## DHCPD配置
### 1、配置iptable转发规则执行如下命令：
```
iptables -F    #清除所有规则来暂时停止防火墙
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -j MASQUERADE    #进行ip伪装

#我这里是内网网段是用的192.168.10.0/24所以我这里-s后跟的参数是192.168.10.0/24
#如果你指定了自己的网段，这里请视自己的情况来定。
```
### 2、调整内核参数，开启内核ip路由转发功能：
```
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p        #使配置生效
```
### 3、安装dhcpd：
`yum install dhcpd -y`
### 4、修改/etc/sysconfig/dhcpd，添加dhcp服务绑定的网卡：
这里我绑定的是网桥设备br0：
```
vim /etc/sysconfig/dhcpd

# WARNING: This file is NOT used anymore.

# If you are here to restrict what interfaces should dhcpd listen on,
# be aware that dhcpd listens *only* on interfaces for which it finds subnet
# declaration in dhcpd.conf. It means that explicitly enumerating interfaces
# also on command line should not be required in most cases.

# If you still insist on adding some command line options,
# copy dhcpd.service from /lib/systemd/system to /etc/systemd/system and modify
# it there.
# https://fedoraproject.org/wiki/Systemd#How_do_I_customize_a_unit_file.2F_add_a_custom_unit_file.3F

# example:
# $ cp /usr/lib/systemd/system/dhcpd.service /etc/systemd/system/
# $ vi /etc/systemd/system/dhcpd.service
# $ ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid <your_interface_name(s)>
# $ systemctl --system daemon-reload
# $ systemctl restart dhcpd.service

DHCPDARGS=br0

:wq  #保存并退出

#其实上面都是废话，只需要最后的一句代码就可以了，或者可以用下面的命令来绑定网桥：
echo "DHCPDARGS=br0" >> /etc/sysconfig/dhcpd
```
### 5、修改配置dhcpd文件：
```
vim /etc/dhcp/dhcpd.conf
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#

option domain-name "tecadmin.net";
option domain-name-servers ns1.tecadmin.net, ns2.tecadmin.net;
default-lease-time 600;
max-lease-time 7200;
authoritative;
#log-facility local7;

subnet 192.168.10.0 netmask 255.255.255.0 {
        option routers                  192.168.10.255;
        option subnet-mask              255.255.255.0;
        option domain-search            "tecadmin.net";
        option domain-name-servers      192.168.10.1;
        range   192.168.10.10   192.168.10.100;
}

#我这里配置的子网段是192.168.10.0/24，可以随意配置， 但是记住，要和上面直接设置的iptables规则在同一网段！

```
### 6、启动dhcpd和配置开机自启动：
```
systemctl start dhcpd
systemctl enable dhcpd
```
### 7、配置防火墙允许dhcpd服务监听udp/67端口：
firewall-cmd --add-service = dhcp --permanent 
firewall-cmd –reload
