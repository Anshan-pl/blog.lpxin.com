---
title: IPv6实现远程访问
date: 2019-09-07 14:20:20
tags: [华硕固件,IPv6,内网穿透,公网IP]
categories: IPv6
---

<center>
IPv6可以实现所有设备具有一个公网<br/>
对于有远程访问，游戏需求，挂TP等都非常方便<br/>
这篇文章中演示通过IPv6开放80端口挂一个web服务并通过域名访问<br/>
</center>
<!--more-->

# 光猫配置
## 开启IPv6
这里先配置好光猫，我这里的环境是移动的免费宽带，使用的是GPON光猫，首先需要注意的是，IP模式中必须要有IPv6否则是没有办法实现IPv6访问的，如下图的设置：
![IPv6](/images/post/ipv6_1.png "IPv6")
我这里是使用的admin账号登陆的，可以问装宽带的师傅要超级管理员密码，或者自己网上寻找破解方法，这里说下我这个型号光猫（HG6201M）破解方式，已破解的可以忽略。

## 开启telnet服务
首先确认你的设备已经和光猫同一子网中，打开网址http://192.168.1.1/cgi-bin/telnetenable.cgi?telnetenable=1 开启光猫的telnet服务。

windows设备也是需要开启telnet，可以通过 控制面板-程序-启动或关闭Windows功能-Telnet Client，勾选上后点击确定即可。
## 获取管理员账户密码
按下Win+R键（Win键就是键盘上windows图标按钮）输入cmd打开电脑命令行，输入telnet 192.168.1.1回车后进入telnet服务
用户名：root密码：hg2x0
进入shell后输入命令：
```
cat /flash/cfg/agentconf/factory.conf
CMCCAdmin
CMCCAdmin7bGfkYs7
```
第一行和第二行分别为用户名和密码。
注意：此方法不一定适用于本型号的所有地区设备，仅参考。

# 路由器配置
这里我使用的是华硕固件使用360 P2改的固件，所有下面的演示全基于华硕固件来说明，实测过PandoraBox和Openwrt一样适用，只是步骤不同。

## IPv6设置
首先进入外部网络（WAN）配置pppoe拨号，这里不用多做介绍了，除了用户名和密码其他都默认就好，进入IPv6设置如下图：
![IPv6](/images/post/ipv6_2.png "IPv6")
所有配置如上图即可，这里我上级运营商分配的DNS有些问题所有我使用了公共IPv6 DNS。设置完成后重启路由器查看 网络地图-外部网络状态页 可以看到IPv6 地址：WAN项已经有值，那么基本就没有问题了，这里可能IPv6可能比较慢，耐心等待几分钟。
PandoraBox和Openwrt的设置基本相同，需要注意的地方是不能勾选上使用内置的DHCPv6管理。

# 测试IPv6
继续测试下该IPv6是否可用，如下图：
![IPv6](/images/post/ipv6_3.png "IPv6")
这里我没有将IPv6地址给打马赛克，其实打不打都没什么意义，因为路由器重启或者地址租约到期之后都会变化。
路由器设置完并且测试IPv6连通性正常后创建一个虚拟机桥接路由器来打开22和80端口：
首先使用ip命令查看获取到的地址：
```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:1f:42:e5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.123.41/24 brd 192.168.123.255 scope global dynamic noprefixroute ens160
       valid_lft 74488sec preferred_lft 74488sec
    inet6 2409:8a30:3213:3210::c46d/128 scope global dynamic noprefixroute
       valid_lft 1528sec preferred_lft 1528sec
    inet6 2409:8a30:3213:3210:9c9d:7cd1:dde2:2640/64 scope global dynamic noprefixroute
       valid_lft 1788sec preferred_lft 1788sec
    inet6 fe80::d279:e545:422:a704/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
获取到IPv6地址后，使用外网设备ping一次测试，如下图：
![IPv6](/images/post/ipv6_4.png "IPv6")
Internet里的设备可以ping通证明该地址是外网地址，接下来将域名绑定到该IPv6地址测试端口是否可以访问：

# 绑定域名
![IPv6](/images/post/ipv6_5.png "IPv6")
可以看到，这里我将二级域名ipv6.lpxin.com的记录值配为了测试了IPv6地址，记录类型是AAAA为IPv6类型。
域名刚设置后需要几分钟的时候才可以正常访问，先尝试配置一个Caddy来监听80端口：


# 打开web服务
```
[root@localhost ~]# curl https://getcaddy.com | bash -s personal
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  7380  100  7380    0     0   5503      0  0:00:01  0:00:01 --:--:--  5503
Downloading Caddy for linux/amd64 (personal license)...
Download verification OK
Extracting...
Putting caddy in /usr/local/bin (may require password)
Caddy v1.0.3 (h1:i9gRhBgvc5ifchwWtSe7pDpsdS9+Q0Rw9oYQmYUTw1w=)
Successfully installed

#首先停止虚拟机上的防火墙，避免防火墙造成的干扰
[root@localhost ~]# systemctl stop firewalld
[root@localhost ~]# echo ':80' >> Caddyfile
[root@localhost ~]# echo 'hello world!' >> index.html
[root@localhost ~]# caddy
Activating privacy features... done.

Serving HTTP on port 80
http://

WARNING: File descriptor limit 1024 is too low for production servers. At least 8192 is recommended. Fix with `ulimit -n 8192`.
```
这里caddy的配置文件只有一行，效果是打开系统下所有地址的80端口，[关于Caddy的使用请参考这篇文档](https://blog.lpxin.com/2019/05/16/Linux%E4%B8%8B%E4%BD%BF%E7%94%A8Caddy%E4%BB%A3%E6%9B%BFNginx%EF%BC%8C%E5%B9%B6%E9%85%8D%E7%BD%AEphp-fpm/)

# 测试端口开放
配置完web服务后，尝试通过ipv6.lpxin.com域名来访问尝试：
![IPv6](/images/post/ipv6_6.png "IPv6")
可以看到我这里的移动没有封IPv6地址的80端口，照这个情况443端口应该也没有封，继续测试ssh的22端口：
```
root@DESKTOP-HV125D3:~# ssh ipv6.lpxin.com
The authenticity of host 'ipv6.lpxin.com (2409:8a30:3213:3210:9c9d:7cd1:dde2:2640)' can't be established.
ECDSA key fingerprint is SHA256:GbHaK7WMZT/YN8t5ltjARUc7NttdCAK3cDtyBIXiu98.
Are you sure you want to continue connecting (yes/no)?
```
22端口也是正常的，低端口都没有封，估计移动IPv6的所有端口都是开放的。这样不论是web服务还是ssh、ftp、windows的远程桌面等都可以通过IPv6实现远程访问控制了，前提是访问的设备同样具有IPv6地址才可以。
