---
title: iptables常用匹配
date: 2019-10-26 14:53:54
tags: [iptables,redhat,netfilter,firewalld,linux]
categories: Iptables
---

<center>
iptables是Linux上常用防火墙工具
但它其实只是一个用户代理工具位于用户空间
而真正的防火墙的安全框架是netfilter位于内核空间
此篇记录工作和日常中使用到的iptables匹配规则
</center>
<!--more-->

# Iptables
不过多介绍iptables，网上原理介绍很多。

## 四表五链
- filter表：负责过滤数据包，包括的规则链有：INPUT、OUTPUT、FORWARD。
- Nat表：涉及网络地址转换（源、目IP；端口），包括的规则链：PREROUTING、POSTROUTING、OUTPUT。
- Mangle表：修改数据包的服务类型、TTL、并且可以配置路由实现QOS，包括规则链有：INPUT、OUTPUT、FORWARD、PREROUTING、POSTROUTING。
- Raw表：决定数据包是否被状态跟踪机制处理，用来作异常处理的一般用不到，包括的规则链：PREROUTING、OUTPUT。


- INPUT链：进来的数据包应用此规则链中的策略，用来过滤进入本机的数据包。
- OUTPUT链：发出的数据包应用此规则链中的策略，处理从本机机发出的数据包。
- FORWARD链：转发数据包时应用此规则链中的策略，处理转发流经本机的数据包。
- PREROUTING链：对数据包作路由选择前应用此链中的规则，所有的数据包进来的时侯都先由这个链处理，其优先级最高且所有数据包都会经过。处理用户请求中的目的地址、端口用来做DNAT，可以作端口转发、ip映射，如：把内网中的80端口映射到路由器的外网端口上。
- POSTROUTING链——对数据包作路由选择后应用此链中的规则，所有的数据包出来的时侯都先由这个链处理，其优先级最低但同样所有数据包都会经过。处理离开本机的数据包，进行源端口 源ip的修改作SNAT：如家用路由器。

## 流程分析
当一个数据包发送至iptables所在主机网卡后，它将首先进入PEROUTING链，内核根据数据包的目的IP地址判断是否进行转发，如果该数据包需要进行转发且内核允许转发则进入流程1.
1. 数据包会经过FORWARD链 -> POSTROUTING链后发送至目的IP

如果该数据包就是需要进入到本机的那么进入流程2.

2. 数据包会接着进入INPUT链，监听该数据包对应端口的进程会收到该包并发送一个回包经过OUTPUT链 -> POSTROUTING链输出。

## 优先级
表间的优先级顺序：
- raw -> mangle -> nat -> filter

链间的优先级顺序：

- 入站数据：PREROUTING -> INPUT
- 出站数据：OUTPUT -> POSTROUTING
- 转发数据：PREROUTING -> FORWARD -> POSTROUTING

链内的匹配优先级顺序

- 链内规则匹配依次进行检查，从上自下找到相匹配的规则后即停止（LOG选项表示记录相关日志）
- 若在对应链中找不到相匹配的规则，则按照该链的默认策略处理（未修改的情况下默认策略为允许ACCEPT，可以使用-P项修改）

# iptables配置文件
```
[root@localhost ~]# cat /etc/sysconfig/iptables
# sample configuration for iptables service
# you can edit this manually or use system-config-firewall
# please do not ask us to add additional ports/services to this default configuration
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```
配置iptables可以直接修改配置文件，在配置文件中添加规则，重启iptables可以立即生效。

# iptables命令语法格式
`iptables [-t 表名] 管理选项 [链名] [匹配条件] [-j 目标动作或跳转]`
注意事项：
- 不指定表名时，默认表示对filter表进行操作
- 不指定链名时，默认表示操作该表内所有链
- 匹配条件是必备参数，除非是修改链的默认策略

# 匹配规则参数
匹配条件：
- 流入、流出接口（-i、-o）
- 来源、目的IP地址（-s、-d）
- 协议类型（-p）
- 来源、目的端口（- -sport、- -dport）

一般指定了来源或目的端口的同时也要指定协议类型，因为端口是TCP、UDP上的概念。
```
-s <匹配来源地址>
-d <匹配目的地址>
#可以是IP、网段、域名，也可以为空（任何地址）
-s 192.168.1.1       匹配来自 192.168.1.1 的数据包
-s 192.168.1.0/24    匹配来自 192.168.1.0 段的所有数据包

-d 172.10.1.0/16     匹配去往 172.10.1.0/16 段的数据包
-d www.baidu.com     匹配去往域名 www.baidu.com 的数据包

--sport <匹配源端口>
--dport <匹配目的端口>
#可以是指定端口，也可以是端口范围
--sport 80           匹配源端口是 80 的数据包
--sport 80:443       匹配源端口是80-443范围内的所有数据包（含80和443端口）

--dport :443         匹配目的端口是443以下的所有数据包（含443端口）
--dport 80:          匹配目的端口是80以上的所有数据包（含80端口）
```

# 查看表内规则
```
[root@localhost ~]# iptables -t filter -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

target：表示匹配到规则后进行的操作
port： 表示需要匹配的协议（tcp、udp、icmp、all）
source：表示源IP
destination：表示目的IP
最后一列是匹配对应的端口：dpt为目的端口、spt为源端口
```
-t参数指定需要进行操作的表，-L查看表内所有规则。

# 添加规则
```
[root@localhost ~]# iptables -t filter -A INPUT -p tcp --dport 80 -j ACCEPT

[root@localhost ~]# iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

-A APPEND 追加一条规则（放在最后），优先级最低
-p 指定匹配协议
--dport 指定匹配的目的端口
-j ACCEPT 对匹配到的数据包进行放行操作
```
表示在filter表中对INPUT链追加一条规则（作为最后一条规则）对所有目的端口是80的数据包执行放行操作

# 插入规则
```
[root@localhost ~]# iptables -t filter -I INPUT 1 -p tcp --dport 443 -j ACCEPT

[root@localhost ~]# iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
-i INSERT 插入一条规则，在指定的链名后面跟上需要插入到的位置，如果不指定位置，那么默认插入到第一条
```
# 删除规则
```
[root@localhost ~]# iptables -L --line-number
Chain INPUT (policy DROP)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https
2    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
3    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination

[root@localhost ~]# iptables -t filter -D INPUT -p tcp --dport 443 -j ACCEPT    #按内容匹配删除

[root@localhost ~]# iptables -t filter -D INPUT 1                               #按序号匹配删除，不论规则内容是什么
```
上面的两条命令效果是一样的，第一条命令是按内容匹配删除，第二条是按规则序号删除。

# 设置链的默认策略（默认规则）
```
[root@localhost ~]# iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

[root@localhost ~]# iptables -t filter -P INPUT ACCEPT

[root@localhost ~]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
链的默认策略是指当一个数据包经过INPUT时没有任何一个规则被匹配到，则按照该链的默认策略处理。
注意：在使用-P参数修改链默认策略时，对应的动作前不能加-j，这也是唯一一种匹配动作前不加-j的情况。

# 清空规则
```
[root@localhost ~]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DROP       tcp  --  anywhere             anywhere             tcp dpt:ssh

[root@localhost ~]# iptables -t filter -F OUTPUT

[root@localhost ~]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
-F 后跟链名则表示清除指定链内的所有规则，如果不指定链名，那么将清除指定表内的所有链规则

```
[root@localhost ~]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DROP       tcp  --  anywhere             anywhere             tcp dpt:ssh

[root@localhost ~]# iptables -t filter -F

[root@localhost ~]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
**这里请特别注意：如果其中某链的默认策略是DROP，那么要谨慎使用-F命令，因为该命令只会清除链内规则，而不会改变链的默认策略，链的默认策略仅受-P控制。如果某链默认策略为DROP，那么-F命令会使远程链接丢失！**

# 清除数据包计数器
```
[root@localhost ~]# iptables -nvL
Chain INPUT (policy DROP 30 packets, 3386 bytes)
 pkts bytes target     prot opt in     out     source               destination
   67  7290 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 62 packets, 9510 bytes)
 pkts bytes target     prot opt in     out     source               destination

[root@localhost ~]# iptables -t filter -Z INPUT

[root@localhost ~]# iptables -nvL
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    7   404 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 4 packets, 400 bytes)
 pkts bytes target     prot opt in     out     source               destination
```
这里也解释下-nvL参数
-L LIST 列出规则：
-v 显示详细信息，包括每条规则的匹配到的数据包数量和匹配到的字节数
-x 在-v的基础上，禁止自动单位换算（K、M）
-n 只显示IP地址和端口号，不显示域名和服务名称
--list-number 显示规则序号

# 端口转发
这里测试一下将访问该iptables的80 web数据包转发到另一台web主机上，网络环境如下：
iptables主机：
ens33: 192.168.123.178

web主机：
ens33: 192.168.123.117

client主机：
windows下的Linux子系统（WSL）eth0：192.168.123.73
```
[root@localhost ~]# iptables -t nat -A PREROUTING -i ens33 -p tcp --dport 80 -j DNAT --to 192.168.123.117:8080

#这里修改源地址有三种写法
[root@localhost ~]# iptables -t nat -A POSTROUTING -d 192.168.123.117 -p tcp --dport 8080 -j SNAT --to 192.168.123.178:8
0
#将发往 192.168.123.117:8080 的数据包源地址修改为192.168.123.178:80

[root@localhost ~]# iptables -t nat -A POSTROUTING -d 192.168.123.117 -p tcp --dport 8080 -j SNAT --to 192.168.123.178
#将发往 192.168.123.117:8080 的数据包源地址修改为192.168.123.178

[root@localhost ~]# iptables -t nat -A POSTROUTING -d 192.168.123.117 -p tcp --dport 8080 -j MASQUERADE
#将发往 192.168.123.117:8080 的数据包源地址修改为本机同网段的地址，适用于动态IP的场景

[root@localhost ~]# iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:192.168.123.117:8080

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  tcp  --  anywhere             192.168.123.117      tcp dpt:webcache
```
注意此时我的iptables主机上的FORWARD链默认规则是DROP，而进入该主机的http请求我们设置了一条转发规则，所以进入该主机的http请求是发不出去的，需要添加一条FORWARD规则对其进行放行：
```
[root@localhost ~]# iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh

Chain FORWARD (policy DROP)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

[root@localhost ~]# iptables -t filter -A FORWARD -d 192.168.123.117 -p tcp --dport 8080 -j ACCEPT

[root@localhost ~]# iptables -t filter -A FORWARD -s 192.168.123.117 -p tcp -j ACCEPT

#使用client端访问测试：
root@DESKTOP-T82P3NR:/mnt/c/Users/Anshan/Desktop# curl 192.168.123.178
hello world</br>web ip: 192.168.123.117:8080
```
这里要提醒一下，因为这里使用了iptables的端口转发，所以需要打开iptables主机上的ip转发功能：
`[root@localhost ~]# echo 1 > /proc/sys/net/ipv4/ip_forward`
这里地址转换参数支持IP地址池，如下：
`iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.1.2-192.168.1.10`
