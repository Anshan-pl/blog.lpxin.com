---
title: RouterOS创建dhcpd和并进行多拨
date: 2019-06-17 19:55:31
tags: [ros,RouterOS,dhcpd,vrrp,多拨]
categories: RouterOS
---

<center>
RouterOS的优势不多说<br/>
这篇教程简单介绍Routeros的安装<br/>
并演示如果创建dhcp server<br/>
和通过vrrp进行伪并发多拨
</center>
<!--more-->

## RouterOS版本
一笔带过，这里直接推荐两个版本：
[Ros 6.42.7-x86](http://www.rosabc.com/thread-68244-1-1.html)
或者直接加下方的Q群，群文件有压缩包。

上面的版本是vm虚拟机或者esxi使用的，而如果是pve或者云服务器想要运行，上面的镜像就不可行了，这里再介绍一个ROS的特殊版本，CHR版本，全称Cloud Hosted Router，从名字就可以看出，它就是为了云而生的，默认开放全功能，可惜端口限速1Mbit，适用于演示和学习。

如果不只是拿来学习，还要鼓捣各种花里胡哨的姿势，那就还需要购买授权了，这个版本的另一个特殊之处在于授权也便宜，不像常规的L系列授权会限制各种用户数之类的，这个版本只限制速率。

## DHCP server创建

### 查找你的LAN口名称

这里假设你已经安装并配置好RouterOS并使用winbox连接上了
点击Interfaces，会出现一个Interface List，这里会显示你这个设备所以的接口，可能有些人不知道哪个是LAN口，哪个是WAN，如果你没有办法对比mac来确认，这里说一个简单的方法，Tx和Rx这两栏里，波动的接口就是你的WAN口，剩下的接口应该就是你的LAN口，因为我不知道你具体的设备。

这里需要记住你的LAN口名称，因为创建DHCP server需要用到，如果有多个LAN口，那就随便选一个。

### 创建一个DHCP server

#### 新建IP-DHCP Server
点击DHCP Setup来新建一个DHCP服务
![ros-1](/images/post/ros/ros-1.png)

#### Interfaces
DHCP Server Interface选择你刚刚查找到的LAN口名称，我这里是ether6，选择好之后，点击Next
![ros-2](/images/post/ros/ros-2.png)

#### Address
DHCP Address Space指定内网段我这里是192.168.10.0/24， 24是子网掩码，代表255.255.255.0，具体请百度，
内网段可以随意，只要不和光猫，或者是上级路由冲突就可以。
![ros-3](/images/post/ros/ros-3.png)

#### Gateway
Gateway for DHCP Network指定内网网关，一般都是.1，或者.254都可以
![ros-4](/images/post/ros/ros-4.png)

#### IP pool
Addresses to Give Out指定内网IP池，ros会通过你给定的IP范围来给内网设备分配IP，一般就是和我一样 .2-.254
![ros-5](/images/post/ros/ros-5.png)

#### DNS server
DNS Servers，填不填都行，不填默认使用上级路由的DNS，填写就用指定的DNS。
![ros-6](/images/post/ros/ros-6.png)

#### Lease Time
Lease Time，dhcp分配的IP租期，默认10分钟，可以改成3天。
![ros-7](/images/post/ros/ros-7.png)

### 修改配置
最后设置好后，继续点击Next，到这里DHCP server就创建好了，但是还需要进行一些修改，才可以用
双击刚刚创建的dhcp1，将Relay中的内容隐藏，如果你这里本来就没有内容，那么就不需要动了，否则你需要改成和我一样。
![ros-8](/images/post/ros/ros-8.png)

### IP-Address
这里可以发现，我们刚刚创建的dhcp1是红色的，这是因为它还没有一个IP，我们需要手动指定一个IP给它。
IP-Address在Address List中点击深蓝色的加号。
![ros-9](/images/post/ros/ros-9.png)

### 给DHCP server分配一个IP
Address填入刚刚设置的IP段，Interface填刚刚DHCP server绑定的LAN口，点击OK后，你会发现，刚刚创建的dhcp1已经由红色变成了黑色，代表它已经可用了。
![ros-10](/images/post/ros/ros-10.png)

### 测试DHCP服务
现在LAN口接入一个设备，测试一下我们刚刚创建的DCHP服务吧。

## "并发"多拨
这里给“并发”打上双引号是因为这里的并发不是真正意义上的pppoe连接请求同时发送给服务器，而是在ROS上同时打开pppoe client，来达到一个伪并发的效果，效果和并发几乎是相差无几的.
我这里介绍的是单线多拨，顾名思义，单宽带实现多拨。

由于我不想传图了，改成了命令行的方式来继续教程，所以这里顺便给出RouterOS的官方文档，方便各位进行查阅：
[RouterOS](https://wiki.mikrotik.com/wiki/Manual:TOC)
### 设置vrrp
ROS的多拨和其他路由系统的多拨方式不同，例如Openwrt是通过macVlan进行多拨的。
我这里为了方便，全部以命令给出设置方法，ROS进入命令行的方法是New Terminal
`/interface vrrp add comment=RP1 interface=ether6 interval=1 name=vrrp1 vrid=1 version=3 v3-protocol=ipv4`
这一步创建了一个vrrp1，其出口接口是ether6，你需要修改interface=ether6为你自己对应的外网接口，分辨内外网接口的方法在上面介绍DHCP的时候已经说了。

### 设置IPv4
当你成功创建了一个vrrp1时，你会发现，这个vrrp1接口是红色的，这是因为你还没有给他分配一个IP地址，所以它当前并不可用。
下面我们来给它一个IPv4地址。
`/ip address add address=10.1.1.1/24 interface=vrrp1`
这里指定的IP是随意的，只要符合规则即可，因为我们几乎用不到这个地址。
命令完成后会发现，vrrp1接口还是红色，我们继续给外网接口，也就是刚刚设置给vrrp的出口接口，我这里为ether6。
`/ip address add address=10.1.1.0/24 interface=ether6`
现在可以看到，vrrp1已经变为了黑色，代表可用了。

### 创建pppoe client
`/interface pppoe-client add add-default-route=no allow=pap,chap,mschap1,mschap2 comment=1 interface=vrrp1 name=pppoe-out1 user=ppp1 password=anshan`
interface=vrrp1表示pppoe客户端对应的接口，也就是刚刚创建的vrrp1，user为运营商给你的用户名，一般都是手机号，或者区号加座机号，password为密码。

### 打开pppoe-out
通过命令行创建的接口是不会立即打开的，需要手动打开，或者通过命令行，图省事，不上图了，直接给出命令。
`/interface pppoe-client enable pppoe-out1`

### 实现伪并发多拨
伪并发多拨很简单，就是让多个pppoe-out接口同时打开即可。
刚刚我们创建了一个vrrp和其对应的pppoe-out，现在让我们再创建一组接口，步骤和上面的完全一样，只需要将名字从1改为2即可，这里唯一一个需要注意的地方是vrrp的参数vrid，这个参数是唯一的，也就是说你再次创建的vrrp接口的vrid需要设置为2，以此类推。不得不提的一点是vrid是可以无限制的，它可以大于254，这也就给我们多拨到三四百，甚至说四五百提供了可能。

我假设你已经创建了两组拨号接口。接下来，我们来进行伪并发多拨。
首先同时选中pppoe-out1和pppoe-out2（按住Ctrl再以此点击out1和out2即可），并保证他们现在都处于禁用状态，再上方界面上的红色叉号代表禁用，接下来让我们同时打开这两个pppoe-out接口，请根据图上的提示操作。
![ros-11](/images/post/ros/ros-11.png)
在我这张图上可以看到，我这里是运行了两个RouterOS的，左侧的ROS是用来进行pppoe client连接的，而右侧的ROS是创建了一个pppoe server供左侧ROS进行多拨测试的。
在这张图右边可以看到，我这里的pppoe server将Only One设置为了no，这个选项代表了是否允许单个用户是否可以多个连接，简单来说，这个选项是多拨的开关，我这里选择了no也就是禁止多拨，但是在log中ppp1用户是进行了双拨，这就是并发多拨的神奇之处，具体原理请百度或Google，这里不多做介绍了。

## 通过IPv6创建vrrp
为什么要给这种方式创建vrrp给一个单独的二级大标题呢，因为这种方式更方便，如果是通过IPv4参数来创建的vrrp则需要给其指定一个IPv4地址（参考3.2. 设置IPv4），而通过IPv6的方式则不需要，如果只是这样，也没什么，而如果你的宽带是可以无限拨的，那么你很可能需要这种方式！因为IPv4创建vrrp是限制在254个，而通过IPv6创建vrrp后，可以再创建254个vrrp，代表着，一共可以同时进行508拨。
介绍完了，直接上命令代码
`/interface vrrp add comment=RP3 interface=ether6 interval=1 name=vrrp3 vrid=3 version=3 v3-protocol=ipv6`
这里假设你创的是第三个vrrp，否则请修改comment、name、vrid这三个参数。

命令成功完成后，你会神奇的发现，vrrp3接口是黑色的，且第一列的状态码是RM，这代表着这个接口是直接可用的，无需再为其指定IP了，接下来，按照3.3. 创建pppoe client继续往下进行即可。

## RouterOS系列教程
接下来，我会继续给出PPPOE Server创建教程，给内网IP指定多拨出口，还有伪并发多拨的脚本等，请持续关注加入收藏哦，谢谢支持。
