---
title: rhel上尝试bond
date: 2019-08-30 08:41:13
tags: [Redhat,rhel,bond,EtherChannel,接口绑定,bonding]
categories: Redhat
---

<center>
在企业及电信Linux服务器环境上<br/>
网络配置都会使用Bonding技术做网口硬件层面的冗余<br/>
防止单个网口应用的单点故障<br/>
且bond应用可以有效的增加带宽
</center>
<!--more-->

# 什么是bonding
Linux bonding 驱动提供了一个把多个网络接口设备捆绑为单个的网络接口设置来使用，用于网络负载均衡及网络冗余
网络接口绑定（或通道绑定）是Linux内核和Red Hat Enterprise Linux支持的技术，允许管理员将两个或多个网络接口组合在一起，形成单个逻辑“绑定”接口，以实现冗余或增加吞吐量。绑定接口的行为取决于模式; 一般而言，模式提供热备用或负载平衡服务。此外，它们还可以提供链路完整性监控。

# bond七种模式
根据官方文档，下图是网络接口绑定的七种模式：


|模式|策略|作用|容错|负载均衡|
|:-:|:-:|:-:|:-:|:-:|
|0|循环策略|传输数据包顺序是依次传输（即：第1个包走eth0，下一个包就走eth1….一直循环下去，直到最后一个传输完毕），此模式提供负载平衡和容错能力；但是我们知道如果一个连接或者会话的数据包从不同的接口发出的话，中途再经过不同的链路，在客户端很有可能会出现数据包无序到达的问题，而无序到达的数据包需要重新要求被发送，这样网络的吞吐量就会下降,此模式需要Switch(交换机)支持和设定|否|是|
|1|主备策略|只有一个设备处于活动状态，当一个宕掉另一个马上由备份转换为主设备。mac地址是外部可见得，从外面看来，bond的MAC地址是唯一的，以避免switch(交换机)发生混乱。此模式只提供了容错能力；由此可见此算法的优点是可以提供高网络连接的可用性，但是它的资源利用率较低，只有一个接口处于工作状态，在有 N 个网络接口的情况下，资源利用率为1/N|是|否|
|2|XOR策略|基于[（源MAC地址与目标MAC地址异或）模数从属计数]进行传输，这为每个目标MAC地址选择其对应的接口，此模式提供负载平衡和容错。|是|是|
|3|广播策略|在所有绑定的接口上传输所有内容，此模式提供容错功能。|是|否|
|4|动态链接聚合|IEEE 802.3ad动态链路聚合，创建共享相同速度和双工设置的聚合组。根据802.3ad规范使用活动聚合器中的所有绑定接口。<br/>先决条件：<br/>- 基本驱动程序中的Ethtool支持，用于检索每个绑定接口的速度和双工。<br/>- 支持IEEE 802.3ad动态链路聚合的交换机。大多数交换机需要某种类型的配置才能启用802.3ad模式。|是|是|
|5|自适应传输负载平衡|不需要任何特殊交换机支持的接口绑定。根据每个接口上的当前负载（相对于速度计算）分配输出流量。当前接口接收传入流量。如果接收接口发生故障，则另一个接口接管故障接口接收的MAC地址。<br/>先决条件：<br/>- 基本驱动程序中的Ethtool支持，用于检索每个从站的速度。|是|是|
|6|自适应负载平衡|包括用于IPV4流量的balance-tlb和接收负载平衡（rlb），并且不需要任何特殊的交换机支持,因为做bonding的这两块网卡是使用不同的MAC地址。在每个slave（接口）上根据当前的负载（根据速度计算）分配外出流量。如果正在接受数据的slave（接口）出故障了，另一个slave接管失败的slave的MAC地址。接收负载均衡是通过ARP协商实现的。绑定驱动程序拦截本地系统在其出路时发送的ARP回复，并使用绑定中的一个接口的唯一硬件地址覆盖源硬件地址，以便不同的对等方使用服务器的不同硬件地址。|是|是|

常用的为0、1、6三种。
# 开启bonding内核模块
```
modprobe --first-time bonding
modinfo bonding
#查看bonding模块信息
```
# 创建bond0
这里我使用的环境是rhel8，默认使用nmcli网络管理工具。
## 查看现有的网络连接
```
nmcli connection show
nmcli connection
nmcli c s
nmcli c
#上面四条命令的回显都是一样的，可以将下面的三条命令理解为第一条命令的简写

NAME                UUID                                  TYPE      DEVICE
Wired connection 1  1298f993-4540-3c63-86da-249387823683  ethernet  ens160
Wired connection 2  5b52a57f-eaee-35d5-b94d-363c382efd1c  ethernet  ens192
Wired connection 3  5ef06105-c23d-3c27-8c34-3106e121df4d  ethernet  ens224
Wired connection 4  db82416b-b6ca-3753-bcba-f0ab28ea3870  ethernet  ens256

#这里我在虚拟机上给出了四条桥接的网络适配器。
```
## 添加一个bond0接口
```
nmcli con add type bond con-name bond0 ifname bond0 mode 6 ip4 192.168.199.66/24
Connection 'bond0' (9e96bd0d-a2c7-4799-a3f5-e9f5d4b60311) successfully added.

nmcli connection show
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  1298f993-4540-3c63-86da-249387823683  ethernet  ens160
Wired connection 2  5b52a57f-eaee-35d5-b94d-363c382efd1c  ethernet  ens192
Wired connection 3  5ef06105-c23d-3c27-8c34-3106e121df4d  ethernet  ens224
Wired connection 4  db82416b-b6ca-3753-bcba-f0ab28ea3870  ethernet  ens256
bond0               9e96bd0d-a2c7-4799-a3f5-e9f5d4b60311  bond      bond0

cat /etc/sysconfig/network-scripts/ifcfg-bond0
BONDING_OPTS=mode=balance-alb
TYPE=Bond
BONDING_MASTER=yes
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=192.168.199.66
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=bond0
UUID=9e96bd0d-a2c7-4799-a3f5-e9f5d4b60311
DEVICE=bond0
ONBOOT=yes

'nmcli con add type bond'命令在/etc/sysconfig/network-scripts目录中创建bond0接口配置文件ifcfg-bond0。
```
上面这个例子创建了名（con-name）为bond0的绑定接口，将接口定义（ifname）为bond0，将模式设置为6即自适应负载平衡模式，并为绑定接口指定IP地址。

## 给bond0绑定从属接口
对于要绑定的每个接口，请使用‘nmcli con add type bond-slave’命令。以下示例将ens192接口添加为绑定从属。该命令不包含con-name参数，因此会自动生成名称。你可以使用con-name参数为从属接口设置名称。
```
nmcli con add type bond-slave ifname ens192 master bond0
Connection 'bond-slave-ens192' (7a776193-f297-42ec-996c-48b9c835fff0) successfully added.
```

以下示例将ens224接口添加为’bond-slave‘：
```
nmcli con add type bond-slave ifname ens224 master bond0
Connection 'bond-slave-ens224' (60210b6a-6c15-40cf-85ed-2e182119a579) successfully added.
```

nmcli connection show命令显示新加入的连接：
```
nmcli connection show
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  1298f993-4540-3c63-86da-249387823683  ethernet  ens160
Wired connection 2  5b52a57f-eaee-35d5-b94d-363c382efd1c  ethernet  ens192
Wired connection 3  5ef06105-c23d-3c27-8c34-3106e121df4d  ethernet  ens224
Wired connection 4  db82416b-b6ca-3753-bcba-f0ab28ea3870  ethernet  ens256
bond0               9e96bd0d-a2c7-4799-a3f5-e9f5d4b60311  bond      bond0
bond-slave-ens192   7a776193-f297-42ec-996c-48b9c835fff0  ethernet  --
bond-slave-ens224   60210b6a-6c15-40cf-85ed-2e182119a579  ethernet  --
```
nmcli con add type bond-slave命令在/etc/sysconfig/network-scripts目录中创建接口配置文件。例如ifcfg-bond-slave-ens192：
```
cat /etc/sysconfig/network-scripts/ifcfg-bond-slave-ens192
TYPE=Ethernet
NAME=bond-slave-ens192
UUID=7a776193-f297-42ec-996c-48b9c835fff0
DEVICE=ens192
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ens224的配置文件除UUID和DEVICE不同，其他配置值都是相同的。
```
## 激活绑定接口
可以使用nmcli命令调出连接，首先启动从属接口，最后再启用bond接口。以下命令为启动从属接口（ens192、ens224）：
```
nmcli connection up bond-slave-ens192
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/6)

nmcli connection up bond-slave-ens224
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)
```

## 启用bond0
```
nmcli connect up bond0
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/11)

#查看bond0状态信息
cat /proc/net/bonding/bond0
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: adaptive load balancing
Primary Slave: None
Currently Active Slave: ens192
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: ens192
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:23:71:18
Slave queue ID: 0

Slave Interface: ens224
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:23:71:22
Slave queue ID: 0
```
每个网络接口都包含在/sys/class/net路径下，也可以通过查看此路径下的接口目录获取信息
```
ls /sys/class/net
bond0  bonding_masters  ens160  ens192  ens224  ens256  lo

cat /sys/class/net/bonding_masters
bond0

ls /sys/class/net/bond0
addr_assign_type  carrier_changes     duplex             link_mode         operstate       queues        uevent
addr_len          carrier_down_count  flags              lower_ens192      phys_port_id    speed
address           carrier_up_count    gro_flush_timeout  lower_ens224      phys_port_name  statistics
bonding           dev_id              ifalias            mtu               phys_switch_id  subsystem
broadcast         dev_port            ifindex            name_assign_type  power           tx_queue_len
carrier           dormant             iflink             netdev_group      proto_down      type

cat /sys/class/net/bond0/operstate
up

cat /sys/class/net/bond0/address
00:0c:29:23:71:18

cat /sys/class/net/bond0/bonding/slaves
ens192 ens224
```
不过基本上这些信息都可以直接通过/proc/net/bonding/bond0文件获取，建议直接使用cat /proc/net/bonding/bond0查看。

## 修改bond模式
```
[root@localhost ~]# nmcli con modify bond0 mode 0
[root@localhost ~]# nmcli con reload bond0
[root@localhost ~]# nmcli con down bond0
Connection 'bond0' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/30)
[root@localhost ~]# nmcli con up bond0
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/33)
```
# 定位多网卡物理位置
通过ethtool命令可以很方便的定位到多网卡环境下故障的链路，因为其效果是使其对应网口的链路灯闪烁，所以可以很方便的确定到故障网口的位置：
```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp3s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether 74:d4:35:0c:d0:b9 brd ff:ff:ff:ff:ff:ff
3: enp1s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:15:17:e4:0d:c2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.105/24 brd 192.168.1.255 scope global dynamic noprefixroute enp1s0f0
       valid_lft 42075sec preferred_lft 42075sec
    inet6 2409:xxxx:xxxx:xxxx::xxx/128 scope global dynamic noprefixroute
       valid_lft 209487sec preferred_lft 123087sec
    inet6 fdcc:9cbd:a968::eb2/128 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fdcc:9cbd:a968:0:23c7:458:850a:abe3/64 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 2409:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx/64 scope global dynamic noprefixroute
       valid_lft 209487sec preferred_lft 123087sec
    inet6 fe80::3142:ccd2:6c93:a337/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
4: enp1s0f1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether 00:15:17:e4:0d:c3 brd ff:ff:ff:ff:ff:ff

#先查看下当前本地环境的网络配置信息，可以看到当前我的系统下有一个板载网口和两个通过pcie网卡扩展的千兆网口。
#因为我是家用PC测试的，也不可能用光纤卡，所以只能用电口网卡进行测试，按原理的话应该都是互通的。

根据ip命令的回显，当前我使用的千兆网卡的其中一个电口enp1s0f0，下面通过ethtool -p和ethtool --identify命令来对该网口进行点亮测试：

ethtool --identify enp1s0f0 20
#点亮enp1s0f0 20秒

ethtool -p enp1s0f0
#使enp1s0f0口长亮，直到按下Ctrl+C结束命令，停止闪烁
```
下面的视频演示了两个点亮网口命令，因为是优酷的视频源，不是https协议所以在大部分浏览器可能会被拦截认定为不安全的内容，如我这里使用的Chrome浏览器，可以通过点击地址栏最后方中小盾牌图标“加载不安全脚本”按钮来查看演示视频：

<iframe height=498 width=510 src='http://player.youku.com/embed/XNDM0MTI5MzA5Ng==' frameborder=0 'allowfullscreen'></iframe>


# 记19/10/18日bond无法up，光模块不发光情况
在一次H3C更新BIOS，启动系统后bond up不起来，且光模块和光纤都有，并单网口也up不起来，多次尝试修复后都无效，最后确定为网卡速率模式问题导致。在系统下使用ethtool下查看千兆单网卡Speed速率为25000Mb/s，这是不正常的，可以使用ethtool命令来关闭网卡的Auto-negotiation自动速率协商，并强制指定网卡的速率即可。
```
[root@localhost ~]# ethtool -s ens38 autoneg off speed 1000
#注意：ethtool命令设置网卡属性后仅是单次生效，重启后就失效了，如果想永久生效那么请使用下列命令:

[root@localhost ~]# echo 'ETHTOOL_OPTS="speed 100 duplex full autoneg off"' >> /etc/sysconfig/network-scripts/ifcfg-ens38
#使用echo命令将配置追加到网卡配置文件即可。
```
## 单网卡启动
在配置完bond后，需要先启动bond的从属网卡后，才可以启动bond生效。
```
[root@localhost ~]# ip link set ens38 up
[root@localhost ~]# ifup ens38 up
#这里使用的是两种命令启动单网卡，当然也可以使用nmcli来启动。

[root@localhost ~]# nmcli con up bond0
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/39)
[root@localhost ~]# nmcli d connect bond0
Device 'bond0' successfully activated with 'd796410e-f4f1-4650-82e3-2fe66ec4c60d'.
#这里启用bond使用的是nmcli命令，这里无所谓使用那条命令来启动单网卡或bond。
```
最后，使用ethtool命令来查看一下接口和bond的信息：
```
[root@localhost ~]# ethtool ens38
Settings for ens38:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Supported pause frame use: No
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  1000baseT/Full
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Speed: 1000Mb/s
        Duplex: Full
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: on
        MDI-X: off (auto)
        Supports Wake-on: d
        Wake-on: d
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: yes
#这里我使用测试环境是虚拟机，所以两张网卡是一样的，可以看到从属接口的speed速率为1000Mb/s，作了bond后如果模式是用带宽聚合效果的那么查看bond接口的话，其速率应该为两个从属接口speed速率之和。

[root@localhost ~]# ethtool bond0
Settings for bond0:
        Supported ports: [ ]
        Supported link modes:   Not reported
        Supported pause frame use: No
        Supports auto-negotiation: No
        Supported FEC modes: Not reported
        Advertised link modes:  Not reported
        Advertised pause frame use: No
        Advertised auto-negotiation: No
        Advertised FEC modes: Not reported
        Speed: 2000Mb/s
        Duplex: Full
        Port: Other
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: off
        Link detected: yes
#可以看到，现在我们使用的bond模式为mode6（balance-alb），有带宽聚合效果，所以bond0的speed速率为2000Mb/s，双从属接口速率之和。
```
