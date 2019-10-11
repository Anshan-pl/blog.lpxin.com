---
title: QEMU进阶操作-使用桥接模式-扩展镜像-使用物理磁盘-创建增量镜像-修改MAC
date: 2019-05-16 12:15:14
tags: [Ubuntu,QEMU,bridge-utils,aarch64,armv8,arm]
categories: QEMU
---
<center>
上一篇文章介绍了基于QEMU的aarch64镜像<br/>
这篇文章将更深入的介绍QEMU的一些用法<br/>
虚拟机桥接网络<br/>
对虚拟机磁盘进行扩展<br/>
使虚拟机挂载物理磁盘<br/>
基础镜像和增量镜像的制作<br/>
虚拟机mac修改
</center>
<!--more-->

## 1、虚拟机网桥实现
### 安装bridge-utils
`apt-get install bridge-utils`
### 创建网桥设备
`sudo brctl addbr br0`
### 配置网桥
这里配置桥接我以Ubuntu系统为例，
如果你是Centos
[那么请看这篇文章里的桥接实现](https://blog.lpxin.com/2019/05/15/centos%E5%8F%8C%E7%BD%91%E5%8D%A1%E9%85%8D%E7%BD%AEdhcpd%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%AE%9E%E7%8E%B0%E5%B1%80%E5%9F%9F%E7%BD%91%E8%AE%BF%E9%97%AE%E5%A4%96%E7%BD%91/ "centos 7配置桥接")
#### 假设你是Ubuntu 16
```
sudo vim /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback

auto ens33
iface ens33 inet manual

auto br0
iface br0 inet dhcp
bridge_ports ens33
bridge_stp off
bridge_fd 0

#重启计算机或
sudo systemctl restart networking / sudo systemctl restart network-manager
#通过ifconfig查看已经有了br0和ens33

```
#### 假设你是Ubuntu 18
```
vim /etc/netplan/50-cloud-init.yaml

#这里的文件名不一定和我相同，请自行确认
#下面是我的网桥配置，可以参考

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
        dhcp4: no
        addresses: [192.168.0.20/24]
        gateway4: 192.168.0.1
        nameservers:
          addresses: [119.29.29.29]

#请根据自己的环境修改配置
#如果不想使用静态IP，那么请删除或注释配置并打开dhcp4: true
#这里不得不吐槽一件事
#dhcp4/dhcp6打开是true而关闭竟然是no
#说好的true/false，yes/no呢！

netplan apply  #使配置生效

#注意,这样会使当前连接的ssh断开。
```
### 创建tap设备
```
ip tuntap add dev tap0 mode tap  #创建一个tap0接口
brctl addif br0 tap0  #在虚拟网桥中增加一个tap0接口
ifconfig tap0 up  #打开tap0接口

#这里的tap设备可以无限多
#修改tap的设备名即可：tapx
```
上面的命令每次重启都需要重新运行一次，可以添加到/etc/rc.local开机自动运行
或者配置为服务

### QEMU使用桥接运行虚拟机
在上一篇文章中已经介绍了如何运行一个Ubuntu-cloud映像
如果你不知道如何进行
[请点击查看这篇文章](https://blog.lpxin.com/2019/05/16/QEMU%E8%B7%A8%E6%9E%B6%E6%9E%84%E4%BB%BF%E7%9C%9Faarch64%E4%BA%91%E6%9C%8D%E5%8A%A1/ "QEMU跨架构仿真aarch64云服务")

```
qemu-system-aarch64 \
    -smp 2 \
    -m 1024 \
    -M virt \
    -cpu cortex-a57 \
    -bios QEMU_EFI.fd \
    -nographic \
    -device virtio-blk-device,drive=image \
    -drive if=none,id=image,file=ubuntu-18.04-server-cloudimg-arm64.img \
    -net nic \
    -net tap,ifname=tap0,script=no,downscript=no
```
使用ifconfig命令查看IP地址确定桥接是否成功

## QEMU扩展镜像容量
### 关闭镜像，扩展镜像容量
`qemu-img resize ubuntu-18.04-server-cloudimg-arm64.img +150G`
这里扩展的大小可以随你指定。同样，可以使用-150G参数来减小磁盘的容量。
值得一提的是，对镜像磁盘进行扩展后，实际占用大小是没有变化的。

### 启动虚拟机，启动命令请参考上一段
### 启动完成后，修改分区表，扩展根目录
```
fdisk -l #使用fdisk命令查看硬盘设备名
Device     Boot Start     End Sectors  Size Id Type
/dev/sda1        8192   93236   85045 41.5M  c W95 FAT32 (LBA)
/dev/sda2       94208 9609215 9515008  4.6G 83 Linux

#这里我没有当时操作的保存了，所以从网上找了一份例子
#你在操作的时候，请根据实际情况修改

fdisk /dev/sda #使用fdisk对sda设备进行分区

#可以输入m来进行查看fdisk的命令，如果想退出可以输入q

Welcome to fdisk (util-linux 2.29.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d #删除分区指令
Partition number (1,2, default 2): 2 #2即sda2分区

Partition 2 has been deleted.

Command (m for help): n #创建新分区
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p #创建主分区
Partition number (2-4, default 2): 2 #创建sda2分区
First sector (2048-11706367, default 2048): 94208 #输入sda2分区起始sector
Last sector, +sectors or +size{K,M,G,T,P} (94208-11706367, default 11706367): 11706367 #默认镜像最后一个sector

Created a new partition 2 of type 'Linux' and of size 5.6 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: N #此时已经扩展成功，选择不删除分区签名

Command (m for help): w #保存此次操作

The partition table has been altered.
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Device or resource busy

#再次使用fdisk -l命令发现分区表修改已经生效，接着调整文件系统大小以填充分区

resize2fs /dev/sda2

```
如果你扩展的磁盘大小大于2T，那么你将无法使用fdisk来对其进行扩容
因为fdisk最大支持到2T，所以我们需要使用parted来对其进行扩容
[你可以点击查看我的这篇文章]( "Fdisk最大只能创建2T分区的盘，超过2T使用parted")

## 虚拟机使用物理磁盘
```
在启动命令行上加入 -hda /dev/sdX 可以加载你的物理磁盘

#设备名根据你的情况修改
```

## QEMU增量镜像制作
在服务器上，经常需要启动数十个或者几十个虚拟机，按照我们现有的方式是安装一个虚拟机，然后复制相应的份数。例如，一个虚机的镜像大小是4G，十个虚机的大小就需要占用40G空间。 
事实上在目前为止里面还没有执行任何程序，这些空间都是分配，实际并不一定都要使用。那么是否能够实现用多少分配多少呢？分析下可以发现，每个虚拟机里面的内核都是一样的，大部分时候我们都不需要去修改里面的内核，是否能够共用内核？ 
Copy-On-Write模式为我们提供了很好的解决方式，通过创建一个基础镜像（base image），里面把各个虚拟机都需要的环境都搭建好，然后基于这个镜像建立起一个个“增量镜像”（增量镜像的初始大小低于1M），每个“增量镜像”对应一个虚拟机，虚拟机对镜像中所有的改变都记录在“增量镜像”里面，基础镜像始终保持不变。这样我们建立十个虚拟机，需要的空间为：4G+10*1M=4G，一下节省了近36G的空间。

对于我们Flexbng的环境，cp/dp的虚机可以共用一个基础镜像，然后各自有自己的增量镜像。 
好处有: 
1）在部署环境时，需要拷贝的文件大小和磁盘占用空间就会降低很多，尤其是多cp/dp的环境。 
2）基础镜像不会被修改，新拉虚机时可以快速创建个“增量镜像”使用

### 清理并压缩基础镜像
清除基础镜像里面的临时文件（例如软件tar包、编译的文件、日志等等），然后退出虚机，并压缩基础镜像，压缩后的镜像为cloud-base.qcow2。
你可以使用qemu-img info 命令来查看前后的区别
`qemu-img convert -c -O qcow2 ubuntu-18.04-server-cloudimg-arm64.img cloud-base.qcow2`
这需要一点时间，请耐心等待

### 创建增量镜像cloud.qcow2
`qemu-img create -f qcow2 -b cloud-base.qcow2 cloud.qcow2`
后面如果想将增量镜像中的修改合入到基础镜像中，需要执行commit命令
`qemu-img commit cloud.qcow2`

## 修改虚拟机的MAC地址
```
-net nic,macaddr=00:1d:92:ab:3f:78

#例如我这里修改后的完整启动命令为

qemu-system-aarch64 \
    -smp 2 \
    -m 1024 \
    -M virt \
    -cpu cortex-a57 \
    -bios QEMU_EFI.fd \
    -nographic \
    -device virtio-blk-device,drive=image \
    -drive if=none,id=image,file=ubuntu-18.04-server-cloudimg-arm64.img \
    -net nic,macaddr=00:1d:92:ab:3f:78 \
    -net tap,ifname=tap0,script=no,downscript=no
```
## KVM模式
上一篇文章已经介绍了如何在KVM模式下运行，如果你是博纳云用户，那么这篇进阶操作可能都是你需要的，和上一篇介绍的一样，启动命令行修改后，即可让虚拟机运行在KVM模式下。
```
qemu-system-aarch64 \
    -enable-kvm \
    -smp 2 \
    -m 1024 \
    -M virt \
    -cpu host \
    -bios QEMU_EFI.fd \
    -nographic \
    -device virtio-blk-device,drive=image \
    -drive if=none,id=image,file=ubuntu-18.04-server-cloudimg-arm64.img \
    -net nic,macaddr=00:1d:92:ab:3f:78 \
    -net tap,ifname=tap0,script=no,downscript=no
```

这篇QEMU进阶操作同样是我回忆结合文档来写的，有不完善或者错误的地方，请留言，感谢你的支持~

我在X86和aarch64架构的开发板(N1)上都进行了测试，树莓派没有测试，因为我手里没有，但操作流程都是一样的，不会有问题，X86没有瓶颈，N1至少六开虚拟机并运行博纳云的计算任务不会造成压力，内存可能会成为你的瓶颈。[内存的问题你可以点击阅读我的这篇文章来解决，因为qemu可以使用swap空间，这里ubuntu-cloud镜像最小指定内存为512，通过修改-m 512](https://blog.lpxin.com/2019/05/16/Linux%E6%89%A9%E5%B1%95swap%E4%BA%A4%E6%8D%A2%E7%A9%BA%E9%97%B4/ "扩展swap")，而网络带宽问题，你可能需要寻求运营商的帮助了。

如果你觉得这篇文章对你有帮助，请点击一下下方的打赏哦，感谢你的支持。
手动感激脸.jpg~
留言人数多的话，我会出N1或者X86的博纳云镜像回馈给大家，所以请一直关注我的这个小站哦~
!["眼神"](/images/post/qemu-jinjie.gif "暗示")

**end**
