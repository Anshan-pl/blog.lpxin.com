---
title: 在LXD容器中运行Kubernetes（K8s）
date: 2019-05-16 16:07:54
tags: [Ubuntu,LXD,LXC,Kubernetes,K8s,kubeadm,kubelet,kubectl]
categories: LXD
---
<center>
LXC为Linux Container的简写<br/>
可以提供轻量级的虚拟化，以便隔离进程和资源<br/>
而且不需要提供指令解释机制以及全虚拟化的其他复杂性<br/>
如果你有一项工作需要大量的虚拟机来完成<br/>
那么LXC将是你的不二之选！
</center>

![赞](/images/post/lxd-k8s.gif "赞")
<!--more-->

## LXD与LXC
什么是LXD呢？这里套用一下官方的介绍
LXD是下一代系统容器管理器。
它提供类似于虚拟机的用户体验，但使用的是Linux容器。

它的图像基于可用于大量Linux发行版的预制图像，
并且是围绕一个非常强大但非常简单的REST API构建的。

很多人可能不是很能理解，但可能听说过老牌容器 LXC（远早于 docker）。
LXD由（Canonical Ltd）和（Ubuntu）开发维护，其灵感可能来自（OpenVZ）等轻量级虚拟机（容器）。
原有的 LXC 工具比较难用（需要用户了解一些底层知识），同时开发团队想要修改（优化）一些默认配置和特性（如安全增强，默认创建非特权容器）。
为了保持兼容性，不宜在旧的已有 LXC 工具（如 lxc-create, lxc-start 等）上动刀，于是新设计封装了一套上层运维操作工具，即 （LXD）。
LXC使用C开发，LXD使用golang开发。早期版本的docker其实也是基于LXC封装，LXD可能也借鉴了docker的一些思想。
LXD 拆分为daemon（命令为 lxd）和客户端（命令为 lxc）两部分。

LXD 的定位很清晰：系统容器，直接对标虚拟机，甚至可以直接运行虚拟机镜像（但是不启动内核）。
系统容器运行整套操作系统（再说一次，除了内核），应用容器（如 docker）运行应用，两者不冲突。
可以在LXD容器里安装和使用docker，跟在物理机和虚拟机上没什么两样。
LXD还支持与OpenStack集成（nova-lxd 项目，可替代 OpenStack 上的虚拟机？）。

LXD远没有docker流行，网上资料不多。
但通过 [LXD官方文档](https://lxd.readthedocs.io/en/latest/?spm=a2c4e.11153940.blogcont578196.16.25b2765cZNID1V "LXD_doc") 和命令行帮助，已经足够轻松了解和使用 LXD了。

## Kubernetes
kubernetes，简称K8s，是用8代替8个字符“ubernete”而成的缩写。是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

## 配置网桥
这里我因为一些特殊的需要，所以不得不让LXD的网络运行在指定的网桥里。
如果你只是为了在LXD中运行kubernetes，那么你可以跳过这一步，继续往下看。
[关于网桥的配置可以点击查看这篇文章的虚拟机网桥实现部分](https://blog.lpxin.com/2019/05/16/QEMU%E8%BF%9B%E9%98%B6%E6%93%8D%E4%BD%9C-%E4%BD%BF%E7%94%A8%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F-%E6%89%A9%E5%AE%B9%E9%95%9C%E5%83%8F-%E5%B9%B6%E5%88%9B%E5%BB%BA%E5%A2%9E%E9%87%8F%E9%95%9C%E5%83%8F/ "桥接模式")，如果你是Ubuntu 18系统,那么可以运行下面的代码来创建并添加网桥，如果不是，或者是Ubuntu16那么请你点击上面的链接来安装配置网桥
### 安装bridge-utils
`apt update && apt install bridge-utils -y`

### 修改网络配置文件
```
# vim /etc/netplan/50-cloud-init.yaml

# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        ens33:
            dhcp4: no
    version: 2

    bridges:
      br0:
        interfaces: [ens33]
        dhcp4: true

#接口名可能与我不同，请根据自己的环境修改.
#需要静态地址配置的，同样移步上面的链接.

# netplan apply
#注意， 这样会使当前连接的ssh断开，使用ifconfig命令可以查看配置是否生效.
# ifconfig
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.131  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 2409:8a30:3212:ec00:2c0d:28ff:feb3:4e32  prefixlen 64  scopeid 0x0<global>
        inet6 2409:8a30:3212:ec00::9af  prefixlen 128  scopeid 0x0<global>
        inet6 fdcc:9cbd:a968::9af  prefixlen 128  scopeid 0x0<global>
        inet6 fe80::2c0d:28ff:feb3:4e32  prefixlen 64  scopeid 0x20<link>
        inet6 fdcc:9cbd:a968:0:2c0d:28ff:feb3:4e32  prefixlen 64  scopeid 0x0<global>
        ether 2e:0d:28:b3:4e:32  txqueuelen 1000  (Ethernet)
        RX packets 213  bytes 22887 (22.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 140  bytes 17597 (17.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:6e:93:7b  txqueuelen 1000  (Ethernet)
        RX packets 42598  bytes 58720485 (58.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10863  bytes 927145 (927.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 171  bytes 15449 (15.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 171  bytes 15449 (15.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## 进行LXD初始化
接下来进行lxd的初始化，我这里的lxd版本是3.0.3,系统是Ubuntu 18，可以使用下面的命令查看版本：
```
# lxd version
3.0.3

# lxd init

Would you like to use LXD clustering? (yes/no) [default=no]: no  #如果你不需要LXD集群，请选no
Do you want to configure a new storage pool? (yes/no) [default=yes]: no  #这里因为我们需要在容器里运行k8s，所以需要dir存储后端，所以我们不使用默认的存储后端，想了解更多的请翻阅官方文档
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: no  #因为我们已经自己创建了网桥，所以这里选no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: yes  #将lxd配置到我们刚刚创建的网桥上，所以选yes
Name of the existing bridge or host interface: br0  #键入我们刚刚创建的网桥名，我这里是br0
Would you like LXD to be available over the network? (yes/no) [default=no]: no  #我们不需要远程管理，所以选no
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] no  #是否自动更新镜像，我这里不需要所以选no，你可以根据自身的情况选择
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no  #无所谓，嫌麻烦选no
```

## 创建LXD存储后端
刚刚在初始化的时候我们没有使用默认的存储后端，因为我们需要dir，所以我们自己创建
```
# lxc storage create lxd-pool dir source=/home/anshan/temp

#这里存储池名称lxd-pool可以自行修改
#source=/xxxxxx也请根据自己的环境进行修改
#这里需要注意一点，LXD的dir存储后端指定目录必须是个空目录，如果不是，请自行修改
#在上一篇介绍扩容的时候我将一个3T大小的磁盘挂载到了/home/anshan/temp路径下，所以这里我就将LXD的存储路径指定到这里吧。
```

## 创建第一个LXD容器
因为LXC官方的镜像源在国外，所以我这里使用清华LXC国内源
当然，你也可以翻墙创建。。。
```
# lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public

#使用清华的源

# lxc image list tuna-images: | grep ubuntu/18.04
| ubuntu/18.04 (7 more)                | a2ce6c4bfd22 | yes    | Ubuntu bionic amd64 (20190515_07:42)         | x86_64  | 122.19MB  | May 15, 2019 at 12:00am (UTC) |
| ubuntu/18.04/arm64 (3 more)          | 3a209c951d60 | yes    | Ubuntu bionic arm64 (20190515_07:44)         | aarch64 | 115.94MB  | May 15, 2019 at 12:00am (UTC) |
| ubuntu/18.04/armhf (3 more)          | 1b4e43c2b431 | yes    | Ubuntu bionic armhf (20190515_07:42)         | armv7l  | 113.02MB  | May 15, 2019 at 12:00am (UTC) |
| ubuntu/18.04/i386 (3 more)           | 582cc1f54c22 | yes    | Ubuntu bionic i386 (20190515_07:42)          | i686    | 123.43MB  | May 15, 2019 at 12:00am (UTC) |
| ubuntu/18.04/ppc64el (3 more)        | 6e44afaff03b | yes    | Ubuntu bionic ppc64el (20190515_07:42)       | ppc64le | 129.92MB  | May 15, 2019 at 12:00am (UTC) |
| ubuntu/18.04/s390x (3 more)          | 5f61242ac150 | yes    | Ubuntu bionic s390x (20190515_07:42)         | s390x   | 119.80MB  | May 15, 2019 at 12:00am (UTC) |

#列出你需要的镜像，因为我这里是想要创建一个ubuntu 18版本的容器，所以我使用了grep命令来进行筛选。

# lxc init tuna-images:a2ce6c4bfd22 ubuntu -s lxd-pool

#如果在这里你遇到了报错Error: not found,请使用 lxc storage list 命令检查你指定的存储后端是否正确

#这里容器名称ubuntu你可以自行修改
#-s lxd-pool是指定后端存储池
#init命令在创建完成之后不会立即启动容器，因为我们还需要对其进行配置使其支持docker和k8s。
#如果你想创建后立即启动，那么你可以使用lxc storage命令来拉取镜像。

# lxc list
+--------+---------+------+------+------------+-----------+
|  NAME  |  STATE  | IPV4 | IPV6 |    TYPE    | SNAPSHOTS |
+--------+---------+------+------+------------+-----------+
| ubuntu | STOPPED |      |      | PERSISTENT | 0         |
+--------+---------+------+------+------------+-----------+

# lxc config edit ubuntu

#加入下面这段参数：
linux.kernel_modules: bridge,br_netfilter,x_tables,ip_tables,ip_vs,ip_set,ipip,xt_mark,xt_multiport,ip_tunnel,tunnel4,netlink_diag,nf_conntrack,nfnetlink,overlay
  raw.lxc: "
    lxc.apparmor.profile = unconfined
    lxc.cgroup.devices.allow = a
    lxc.mount.auto = proc:rw sys:rw cgroup:rw
    lxc.cap.drop = "
  security.nesting: "true"
  security.privileged: "true"


#你可以参考我这里完整的参数，因为这段代码需要插入到指定的位置，否在在运行容器时会报错：
architecture: x86_64
config:
  image.architecture: amd64
  image.description: Ubuntu bionic amd64 (20190515_07:42)
  image.os: Ubuntu
  image.release: bionic
  image.serial: "20190515_07:42"
  linux.kernel_modules: bridge,br_netfilter,x_tables,ip_tables,ip_vs,ip_set,ipip,xt_mark,xt_multiport,ip_tunnel,tunnel4,netlink_diag,nf_conntrack,nfnetlink,overlay
  raw.lxc: "
    lxc.apparmor.profile = unconfined
    lxc.cgroup.devices.allow = a
    lxc.mount.auto = proc:rw sys:rw cgroup:rw
    lxc.cap.drop = "
  security.nesting: "true"
  security.privileged: "true"
  volatile.apply_template: create
  volatile.base_image: a2ce6c4bfd228c813313c97a8ca31854402a52dd066170938dffa0b76155e8e3
  volatile.eth0.hwaddr: 00:16:3e:04:ea:f7
  volatile.idmap.base: "0"
  volatile.idmap.next: '[]'
  volatile.last_state.idmap: '[{"Isuid":true,"Isgid":false,"Hostid":100000,"Nsid":0,"Maprange":65536},{"Isuid":false$devices:
  root:
    path: /
    pool: lxd-pool
    type: disk
ephemeral: false
profiles:
- default
stateful: false
description: ""

#注意，这里LXD修改容器配置的默认编辑器是nano，文件格式为yaml，请遵循yaml语法
#Ctrl+o  -保存热键
#Ctrl+x  -退出热键

```

## 运行LXD容器
```
lxc start ubuntu  #启动容器
lxc exec ubuntu bash  #可以理解为进入容器bash
apt update && apt install net-tools iputils-ping iproute2 gnupg wget curl -y  #在容器中安装一些常用的工具
```
到这里就结束了，你可以尝试安装并运行docker~~和Kubernetes~~
忘记了，如果你是想运行Kubernetes，那么你还有一个配置文件需要cp，请你继续向下阅读。

## 修改LXD容器的MAC
这里继续给出一个LXD比较常用的命令吧，尤其对于博纳云的用户来说
`lxc config set container_name volatile.eth0.hwaddr yo:ur:ma:ca:dd:ress`
这里的container_name为你的LXD容器名，比如我演示这里就为ubuntu
假设你的网络接口名为eth0，后面为你需要的mac地址，请自行修改

如果你是博纳云用户，想尝试继续运行X86的脚本，那么请你继续向下阅读

## 更新系统内核
请注意！如果你是博纳云用户，那么请先跳过这一步，看下方的 博纳云用户安装X86脚本。
如果不是博纳云用户，你也需要先在LXD容器内安装好docker后再继续更新内核！

当你创建好第一个LXD容器之后，你可以正常安装Kubernetes，但是可能无法正常使用。
因为目前不管在什么版本的kernel上都有一个bug（包括目前最新的kernel 5.1.2版本上）
这是br_netfilter内核模块中的内核错误（或缺少的功能）。
我不知道为什么kernel没有修复这个错误，至少是在上游的版本里。

但是好消息是在github上已经有人尝试修复了这个错误。
[具体请查看这里](https://github.com/brauner/linux/commit/91094df1183e80d75c89a7bb088d922d338bfa88 "4.19")

错误是在源码上进行修复的，所以我们首先需要对其进行编译安装。
[可以点击查看我之前关于kernel编译安装的文章](https://blog.lpxin.com/2019/05/15/%E4%BD%BF%E7%94%A8make-kpkg%E7%BC%96%E8%AF%91deb%E5%86%85%E6%A0%B8%E5%AE%89%E8%A3%85%E5%8C%85/ "编译安装kernel")
如果你不想对其进行编译，那么好消息是我已经将它们编译为了deb，你可以直接下载安装。
但是这需要你的系统是Debian系列。[请点击这里进行下载](https://blog.lpxin.com/temp/4.19_kernel.tar.gz "4.19_kernel"),或直接运行下面的命令即可。
~~在使用这个修复过的内核版本后，目前我没有发现系统任何的异常。~~
![尴尬](/images/post/lxd-k8s3.jpg "尴尬")

我在vm15.1版本遇到了一个内核错误问题，目前我不知道应该怎么解决，下面我会提到，但是如果你使用的是pve或者物理机，那么你将不会遇到这个问题。vm用户也不用担心，因为如果你是按照我的文章顺序来进行安装的，那么你已经成功避开了这个错误。

这里我假设你是通过我编译好的kernel进行的，如果不是，那么你需要自行编译安装此内核
```
# wget https://blog.lpxin.com/temp/4.19_kernel.tar.gz
# tar xzvf 4.19_kernel.tar.gz
# cd 4.19_kernel
# dpkg -i *.deb
# reboot
# uname -a

Linux ubuntu18 4.19.0-rc7-brauner-brnf #22 SMP Sun Oct 28 21:02:38 UTC 2018 x86_64 x86_64 x86_64 GNU/Linu
```
如果这里你的内核也为4.19.0-rc7，那么恭喜你已经成功安装了修复后的内核。

## 博纳云用户安装X86脚本
LXD的优势不言而喻，相较于虚拟机，LXD的资源占用成本几乎可以忽略不记，因为它和主机共享内核，而不重新运行一个新的内核。
博纳云计算任务有NAT类型和硬盘大小的限制，通过LXD的DIR存储后端，和主机共享存储空间，比用虚拟机指定存储空间大小要方便的多，而且省去了不必要的硬盘成本，这里不得不说一下，如果你的根目录大小大于120GB，那么你完全无须再加一个移动硬盘或者U盘给博纳云，因为后台会认你的根目录大小，非常方便。更低的资源占用意味着更少的电费成本，这对于一些用户来说，应该很重要,而对于内存，同样和主机共享，所有内存共享，且大小就为总内存大小。

而网络NAT类型要求，我们通过之前指定的网桥br0，即可实现桥接到主机的上一级路由，NAT类型随意调整，你可以移步下方官方后台成果展示。
### 下载X86脚本
跟着官方的方法走一遍
`wget https://raw.githubusercontent.com/qinghon/BonusCloud-Node/master/x86_64/install.sh -O install.sh&&sudo bash install.sh`
这里因为LXD的网络接口名称默认为eth0，所有不需要修改网络接口名或修改install.sh，直接运行就好
静等脚本运行结束。

脚本运行结束后，因为之前我们跳过了 更新系统内核 这一步，现在我们来操作,注意，一定要确定官方的X86脚本已经运行完成，且没有报错后，再继续运行下面的代码 更新系统内核！
```
# exit  #先退出容器的shell
# lxc stop ubuntu  #停止容器
# wget https://blog.lpxin.com/temp/4.19_kernel.tar.gz  #在宿主机内安装修复后的新内核，因为容器内你没有权限，哪怕显示是root
# tar xzvf 4.19_kernel.tar.gz
# cd 4.19_kernel
# dpkg -i *.deb
# reboot
# uname -a

Linux ubuntu18 4.19.0-rc7-brauner-brnf #22 SMP Sun Oct 28 21:02:38 UTC 2018 x86_64 x86_64 x86_64 GNU/Linu
```
如果这里你的内核也为4.19.0-rc7，那么恭喜你已经成功安装了修复后的内核。

### 执行绑定命令
因为我们刚刚更新了内核，所有这个时候容器可能在stop状态，我们可以先使用lxc start ubuntu命令来运行容器，也可能会提示容器已经运行，这没有关系
```
# lxc start ubuntu
# lxc exec ubuntu bash
# curl -H "Content-Type: application/json" -d '{"bcode":"xxxx-xxxxxxxx","email":"xxxx@xxxx"}' http://localhost:9017/bound
#请根据自己的bcode和email修改
```
### 运行Kubernetes检测
```
# kubeadm join computemaster.bxcearth.com:6443 --token wl0fa4.l4x336ln2t71s26w --discovery-token-ca-cert-hash sha256:a43579a8e9e9f8042ffe16858afe897aeec9f6ad823c82de8b39eb20d15746d1 --ignore-preflight-errors=SystemVerification --node-name=0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed

[WARNING SystemVerification]: failed to parse kernel config: unable to load kernel module "configs": output - "modprobe: ERROR: ../libkmod/libkmod.c:586 kmod_search_moddep() could not open moddep file '/lib/modules/4.19.0-rc7-brauner-brnf/modules.dep.bin'\nmodprobe: FATAL: Module configs not found in directory /lib/modules/4.19.0-rc7-brauner-brnf\n", err - exit status 1
        [WARNING Hostname]: hostname "0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed" could not be reached
        [WARNING Hostname]: hostname "0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed" lookup 0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed on 127.0.0.53:53: server misbehaving
[preflight] Some fatal errors occurred:
        [ERROR RequiredIPVSKernelModulesAvailable]: error getting required builtin kernel modules: exit status 1(cut: /lib/modules/4.19.0-rc7-brauner-brnf/modules.builtin: No such file or directory
)
        [ERROR Swap]: running with swap on is not supported. Please disable swap
[preflight] If you know what you are doing, you can make a check non-fatal with --ignore-preflight-errors=...
```
这是博纳云加入集群的参数，如果你不是博纳云用户，那么你可以用使用自己的方法来进行测试，或者使用kubeadm init。
如果你只是单纯的想使用LXD来多挂博纳云，那么你可以忽略所有的参数
只需要更改--node-name=0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed这里的bcode换成你自己的就可以了
这里你也可以不更换参数，但是这样的话，你这里加入到集群的将会是例子中的bcode。
这也算是博纳云不是bug的bug，用户可以自己指定加入集群的bcode，而不用是绑定命令的码

废话了这么多，我们来看正题，这里忽略所有的警告，我们只看2个ERROR：
这里如果你有3个报错，且第一个报错信息为/proc/sys/net/bridge folder missing inside of container, even with kernel modules enabled，那么请相信我，你的内核没有更新成功，至少没有通过修复后的内核启动系统，请自行查找原因，或者在下方的评论区留言，我会随时查看。

+ 获取不到内核模块信息，这点很好理解，因为我们是在LXD容器内运行了K8s，LXD的安全策略使得容器无法获取到模块文件，但是这里K8s只是单纯了检测文件是否存在。那么我们可以通过copy宿主机上的内核模块文件来绕过检测，这里因为容器是桥接的，所有你需要获取一下宿主机的IP，不用多说怎么获取了吧，ifconfig即可，接着我们复制内核模块文件到容器
```
mkdir -p /lib/modules/4.19.0-rc7-brauner-brnf
scp anshan@192.168.1.151:/lib/modules/4.19.0-rc7-brauner-brnf/modules.builtin /lib/modules/4.19.0-rc7-brauner-brnf/modules.builtin
```
好了，就是这么简单，第一个ERROR我们已经解决了，那么继续第二个
+ 这个错误就更简单了，他的报错已经说明了一切：不支持在交换空间打开的情况下运行。请禁用交换空间，这里只有一点需要注意，LXD容器是和宿主机共享硬件资源的，但LXD容器并没有对硬件操作的权限，所有我们需要在宿主机中关闭所有交换分区
```
exit  #退出容器的shell
swapoff -a  #关闭所有交换分区
lxc exec ubuntu bash
```
到了这里，K8s所有的报错已经解决了，我们可以重新运行kubeadm join xxx命令来检测是否可以正常加入集群
如果你的报错和我一样，只有这两个ERROR，那么我可以自信的告诉你，不必去看了，耐心等待博纳云后台的刷新吧
博纳云后台刷新应该在10-30min左右

### Kubernetes重置
如果你在解决了两个报错后，又再次测试了K8s，那么你应该得到如下输出
```
        [WARNING SystemVerification]: failed to parse kernel config: unable to load kernel module "configs": output - "modprobe: ERROR: ../libkmod/libkmod.c:586 kmod_search_moddep() could not open moddep file '/lib/modules/4.19.0-rc7-brauner-brnf/modules.dep.bin'\nmodprobe: FATAL: Module configs not found in directory /lib/modules/4.19.0-rc7-brauner-brnf\n", err - exit status 1
        [WARNING Hostname]: hostname "0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed" could not be reached
        [WARNING Hostname]: hostname "0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed" lookup 0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed on 127.0.0.53:53: server misbehaving
[discovery] Trying to connect to API Server "computemaster.bxcearth.com:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://computemaster.bxcearth.com:6443"
[discovery] Requesting info from "https://computemaster.bxcearth.com:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "computemaster.bxcearth.com:6443"
[discovery] Successfully established connection with API Server "computemaster.bxcearth.com:6443"
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.12" ConfigMap in the kube-system namespace
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[preflight] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "0307-4f16c18a-5b75-40db-ba38-dff92fbc5fed" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```
如果你得到了如上输出，那么你的K8s已经可以正常运行了，但是你会发现你的这个容器已经加入了集群，该如果重置，或者说是初始化呢，不用担心，运行下面的代码即可
```
kubeadm reset

[reset] WARNING: changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] are you sure you want to proceed? [y/N]:y  #这里kubeadm给出了一个警告，告诉你继续操作将会还原k8s，默认是N，请键入y回车以继续


[preflight] running pre-flight checks
[reset] stopping the kubelet service
[reset] unmounting mounted directories in "/var/lib/kubelet"
E0517 00:44:36.961399   15615 reset.go:140] [reset] failed to unmount mounted directories in /var/lib/kubelet:
[reset] no etcd manifest found in "/etc/kubernetes/manifests/etcd.yaml". Assuming external etcd
[reset] please manually reset etcd to prevent further issues
[reset] deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes]
[reset] deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
```
好了，这个容器里的K8s已经重置了，耐心等待博纳云自动加入集群并刷新后台吧。


### 复制容器，实现多挂
假设你已经完成了上述的操作，现在你需要对容器进行多次复制进行多挂，但是你复制的每个容器都是绑定过bcode的，所以，我们可以先对容器进行一下“清理”
```
rm /opt/bcloud/ca.crt /opt/bcloud/client.crt /opt/bcloud/client.key
kubeadm reset

[reset] WARNING: changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] are you sure you want to proceed? [y/N]:y  #这里kubeadm给出了一个警告，告诉你继续操作将会还原k8s，默认是N，请键>
入y回车以继续


[preflight] running pre-flight checks
[reset] stopping the kubelet service
[reset] unmounting mounted directories in "/var/lib/kubelet"
E0517 00:44:36.961399   15615 reset.go:140] [reset] failed to unmount mounted directories in /var/lib/kubelet:
[reset] no etcd manifest found in "/etc/kubernetes/manifests/etcd.yaml". Assuming external etcd
[reset] please manually reset etcd to prevent further issues
[reset] deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kuberne
tes]
[reset] deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf
/etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
```
好了，现在ubuntu容器的清理已经结束，它现在是一个“干净的”博纳云容器，我们将这个容器多次复制，实现多挂
```
lxc stop ubuntu
lxc cp ubuntu bxc-00
lxc cp ubuntu bxc-01
lxc cp ubuntu bxc-xx
lxc list
+--------+---------+------+------+------------+-----------+
|  NAME  |  STATE  | IPV4 | IPV6 |    TYPE    | SNAPSHOTS |
+--------+---------+------+------+------------+-----------+
| bxc-00 | STOPPED |      |      | PERSISTENT | 0         |
+--------+---------+------+------+------------+-----------+
| bxc-01 | STOPPED |      |      | PERSISTENT | 0         |
+--------+---------+------+------+------------+-----------+
| ubuntu | STOPPED |      |      | PERSISTENT | 0         |
+--------+---------+------+------+------------+-----------+
```
你需要挂多少个码，就复制多少个容器，这里的ubuntu就可以当成一个父容器，复制出来的bxc-xx都是子容器，你以后所有的挂机任务都交给bxc容器就可以了，ubuntu容器可以暂时忘记它，只在你需要创建新的bxc容器时记起，进行copy操作


现在我们进入一个bxc容器，演示一下绑码操作
```
lxc start bxc-00
lxc exec bxc-00 bash
curl -H "Content-Type: application/json" -d '{"bcode":"xxxx-xxxxxxxx","email":"xxxx@xxxx"}' http://localhost:9017/bound
exit
```
对，就是这么简单，甚至你可以直接启动容器后使用lxc list命令查看到容器的IP，接着使用lxc exec bxc-00 /bin/sh -c ""命令来进行绑码操作

## 成果展示
这里注意一下的是，如果你在短时间内大量绑码，那么你可能需要等待一段时间，给容器一些反应时间，具体试机器配置和网络环境而定，下面我给出我这里的lxc list列表，因为我这里只做演示，所有我只运行了4个bxc容器，不要怀疑为什么是运行4个容器，因为我的账号上只有4个码了。。。
```
lxc list

+--------+---------+----------------------+-----------------------------------------------+------------+-----------+
| bxc-01 | RUNNING | 192.168.1.157 (eth0) | fdff:4243:4c4f:5544:3000:1030:14:16ef (tun0)  | PERSISTENT | 0         |
|        |         | 172.17.0.1 (docker0) | fdcc:9cbd:a968::428 (eth0)                    |            |           |
|        |         | 10.8.6.241 (tun0)    | fdcc:9cbd:a968:0:216:3eff:feb5:b707 (eth0)    |            |           |
|        |         |                      | 2409:8a30:3212:ec00::428 (eth0)               |            |           |
|        |         |                      | 2409:8a30:3212:ec00:216:3eff:feb5:b707 (eth0) |            |           |
+--------+---------+----------------------+-----------------------------------------------+------------+-----------+
| bxc-02 | RUNNING | 192.168.1.158 (eth0) | fdff:4243:4c4f:5544:3000:1030:14:16f0 (tun0)  | PERSISTENT | 0         |
|        |         | 172.17.0.1 (docker0) | fdcc:9cbd:a968::d92 (eth0)                    |            |           |
|        |         | 10.8.6.242 (tun0)    | fdcc:9cbd:a968:0:216:3eff:fe76:65e0 (eth0)    |            |           |
|        |         |                      | 2409:8a30:3212:ec00::d92 (eth0)               |            |           |
|        |         |                      | 2409:8a30:3212:ec00:216:3eff:fe76:65e0 (eth0) |            |           |
+--------+---------+----------------------+-----------------------------------------------+------------+-----------+
| bxc-03 | RUNNING | 192.168.1.156 (eth0) | fdff:4243:4c4f:5544:3000:1030:17:16d1 (tun0)  | PERSISTENT | 0         |
|        |         | 172.17.0.1 (docker0) | fdcc:9cbd:a968::651 (eth0)                    |            |           |
|        |         | 10.8.6.211 (tun0)    | fdcc:9cbd:a968:0:216:3eff:fedb:22f2 (eth0)    |            |           |
|        |         |                      | 2409:8a30:3212:ec00::651 (eth0)               |            |           |
|        |         |                      | 2409:8a30:3212:ec00:216:3eff:fedb:22f2 (eth0) |            |           |
+--------+---------+----------------------+-----------------------------------------------+------------+-----------+
| bxc-04 | RUNNING | 192.168.1.155 (eth0) | fdff:4243:4c4f:5544:3000:1030:14:16f1 (tun0)  | PERSISTENT | 0         |
|        |         | 172.17.0.1 (docker0) | fdcc:9cbd:a968::c4a (eth0)                    |            |           |
|        |         | 10.8.6.243 (tun0)    | fdcc:9cbd:a968:0:216:3eff:fe6b:319a (eth0)    |            |           |
|        |         |                      | 2409:8a30:3212:ec00::c4a (eth0)               |            |           |
|        |         |                      | 2409:8a30:3212:ec00:216:3eff:fe6b:319a (eth0) |            |           |
+--------+---------+----------------------+-----------------------------------------------+------------+-----------+
```
然后我们等待半小时，继续放出博纳云后台截图
![admin](/images/post/lxd-k8s4.png "admin")

到这里也就全部结束啦，我们已经实现了通过LXD实现kubernetes的运行，理论上这在arm机器上也是可以的，你不用怀疑，因为我已经尝试过了，它很正常的运行了博纳云计算任务。

如果你想要N1的多挂，那么你可以通过这篇文章里的LXD实现多挂，[也可以点击查看这篇文章，通过QEMU-KVM来实现多挂](https://blog.lpxin.com/2019/05/16/QEMU%E8%B7%A8%E6%9E%B6%E6%9E%84%E4%BB%BF%E7%9C%9Faarch64%E4%BA%91%E6%9C%8D%E5%8A%A1/ "QEMU")，N1的性能不会造成瓶颈，我已经测试了，因为它运行在KVM模式下！

如果你觉得这篇文章对你有帮助，那么可以打赏点零花钱嘛~ <br/>!\[手动感激脸.jpg](/images/post/感激.jpg "感谢打赏")
欢迎大家进行评论留言，如果留言反应的人比较多，那么我会尝试制作包括但不限于vm，esxi，pve甚至是arm架构比如N1的镜像。

这里我已经创建了一个QQ群，进行技术讨论，如果你遇到问题，或者大佬请加一下，感谢支持哈~
群号：702993923
![qq-qun](/images/post/qq-code.jpg "qq-qun")

下篇文章我会介绍通过软路由和硬路由实现Full Cone（NAT1）模式
后期如果博纳云用户较多，我会给出LXD一键复制，绑定脚本,所以，请多多关注本站哦~
![给点零花钱](/images/post/lxd-k8s2.gif "get money")

**end**
