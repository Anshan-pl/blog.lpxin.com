---
title: QEMU跨架构仿真aarch64云服务
date: 2019-05-16 10:41:04
tags: [Ubuntu,QEMU,arm64,ubuntu cloud,aarch64,armv8,bridge-utils,tun/tap]
categories: QEMU
---
<center>
对于想尝试ARM64硬件，但又不确定自己是否真的需要<br/>
那么你可以在你的X86 Linux计算机上运行64位ARM代码<br/>
Ubuntu cloud是官方的预装磁盘映像，我们可以尝试仿真运行它<br/>

![这对一些人来说，非常值得期待](/images/post/qemu-cloud.jpeg "享受吧")
</center>
<!--more-->

## 1、安装QEMU 3.1
### 1、下载QEMU 3.1源码
```
#这里我使用的是编译安装的方法
#不从源仓库apt install安装的原因是
#官方仓库里的QEMU版本过低，会遇到一些奇怪的问题

wget https://download.qemu.org/qemu-3.1.0.tar.xz
tar xvJf qemu-3.0.0.tar.xz
cd qemu-3.0.0
```
### 2、配置QEMU编译环境
#### 假设你是Debian系列：
`apt install libglib2.0-dev libtool libpixman-1-dev flex bison -y`
#### 假设你是Redhat系列:
`yum install zlib-devel glib2-devel pixman-devel -y`

### 3、编译安装QEMU 3.1
```
./configure --target-list=aarch64-softmmu
#因为这里我们是仿真aarch64，所以选择只编译qemu-system-aarch64模块
#如果你想要了解更多，那么请直接运行./configure
make -j4
sudo make install
```
编译安装成功后运行qemu-system-aarch64你将会看到输出
如果你是Redhat系列平台，那么你需要把qemu-system-aarch64程序加入到环境变量，或者以绝对路径运行
## 2、UEFI固件下载
以前系统的启动过程可以简化为 BIOS固件—->引导程序—->操作系统，但是由于传统的BIOS启动方式存在许多问题，如bios运行在16位模式，寻址空间小，运行慢等，所以现在X86、ARM架构等架构都改采用了改进的 UEFI 启动方式（当然会有兼容传统BIOS启动方式的考虑），这种情况下系统启动过程如下图所示。 
 
上图启动过程详细我也不太清楚，大家可以看看wiki上的进一步介绍，这里需要说明的是，UEFI启动中最开始执行的也是专门的UEFI固件。因此，我们要想引导到安装光盘（支持UEFI模式）进一步安装aarch64架构的系统，先要下载对应架构（这里是aarch64）的UEFI固件。 
```
mkdir temp && cd temp
wget http://releases.linaro.org/components/kernel/uefi-linaro/16.02/release/qemu64/QEMU_EFI.fd
```

## 3、下载Ubuntu cloud磁盘映像
注意，如果这里你不想使用云映像，而是希望使用Ubuntu arm安装文件
[那么请你到这里查看](https://blog.csdn.net/chenxiangneu/article/details/78955462 "qemu run aarch64")
```
wget https://cloud-images.ubuntu.com/releases/bionic/release/ubuntu-18.04-server-cloudimg-arm64.img

#或许这里你会下载的很慢，可以尝试找一找国内源
#或者翻墙哦
```
## 4、 自定义启动映像
云图像很简单-没有用户设置，没有默认用户组合（这也是我使用cloud图像而不是iso安装的原因，省时省力，也省资源），因此要登录图像，我们需要在首次启动时自定义图像。用于此的工具是cloud-init。使用cloud-init的最简单方法是使用设置文件传递块媒体 - 当然，对于真正的云部署，您可以使用基于网络的初始化协议cloud-init支持。
### 确定您当前的用户名并获取当前的ssh公钥：
```
root@ubuntu:~/temp# whoami
root
root@ubuntu:~/temp# cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC...

#如果这里你的回显是未找到文件，那么你需要先生成一对密钥
#请尝试运行下面的命令

ssh-keygen -t rsa -P ''
#-P表示密码，-P '' 就表示空密码，也可以不用-P参数，这样就要三车回车，用-P就一次回车。
#这样，你再次运行cat ~/.ssh/id_rsa.pub，将会得到公钥

```
### 创建cloud.txt
```
vim cloud.txt
#cloud-config
users:
  - name: root  #这里替换为你当前的用户名
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC...  #这里替换为你刚刚获得的公钥
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash

#重要提示：#cloud-config不是注释，没有它就会无声地失败。
```
### 配置映像
```
apt install cloud-utils -y
cloud-localds --disk-format qcow2 cloud.img cloud.txt

#默认情况下cloud-localds会创建一个原始图像，QEMU现在抱怨不得不猜测这样的图像
#因此--disk-format qcow2用来指定QEMU格式可以防止产生警告
```

## 5、启动Ubuntu-cloud映像
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
    -device virtio-blk-device,drive=cloud \
    -drive if=none,id=cloud,file=cloud.img \
    -device virtio-net-device,netdev=user0 \
    -netdev user,id=user0 \
    -redir tcp:2222::22
```

### 参数解释
-smp 2 - 2个（虚拟）核心。
-m 1024 - 1024MB的系统内存。
-M virt - 模拟通用QEMU ARM机器。
-cpu cortex-a57 - 要模拟的CPU模型。
-bios QEMU_EFI.fd - 要使用的BIOS固件文件。
-nographic - 输出到终端（而不是打开一个支持图形的窗口）。
-device virtio-blk-device,drive=image - 创建一个名为“image”的Virtio块设备。
-drive if=none,id=image,file=ubuntu-16.04-server-cloudimg-arm64-uefi1.img - 使用“图像”设备和我们的云服务器磁盘映像创建驱动器。
-device virtio-blk-device,drive=cloud - 创建另一个名为“云”的Virtio块设备。
-drive if=none,id=cloud,file=cloud.img - 使用“云”设备和我们的云配置磁盘映像创建驱动器。
-device virtio-net-device,netdev=user0 - 创建一个名为“user0”的Virtio网络设备
-netdev user,id=user0 - 使用设备“user0”创建用户模式网络堆栈
-redir tcp:2222::22 - 将主机上的端口2222映射到guest虚拟机上的端口22（标准ssh端口）。
### qemu-systemctl-aarch64允许仿真的框架
在这里，我们创建了一个通用的QEMU ARM机器。你可以看到可能的ARM机器的完整列表，如下所示：
```
qemu-system-aarch64 -M help
akita                Sharp SL-C1000 (Akita) PDA (PXA270)
...
z2                   Zipit Z2 (PXA27x)
```
这个列表似乎包括所有ARM机器，而不仅仅是64位机器。最新版本的QEMU（但不是目前Ubuntu 16.04 LTS附带的版本）包括众所周知的Raspberry Pi 2（但不是3）。
对于给定的机器，你可以看到支持的处理器：
```
qemu-system-aarch64 -M virt -cpu help
 arm1026
 ...
 ti925t
```
一旦你运行上面的命令以启动仿真的ARM64计算机，它将需要几分钟的启动时间，并将输出如下内容：
```
error: no suitable video mode found.
EFI stub: Booting Linux Kernel...
EFI stub: Using DTB from configuration table
EFI stub: Exiting boot services and installing virtual address map...
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Initializing cgroup subsys cpuset
```
关于“找不到合适的视频模式”的初始错误可以忽略 - 我们专门设置-nographic。
最终会出现一个登录提示 - 这不能用在我们的cloud-config文件中，我们只指定了基于密钥的ssh登录。
在出现登录提示后，将显示各种作业（在引导过程中启动）运行的进一步输出的速度。
第一次启动给定系统时，你应该看到输出确认已安装上面指定的ssh密钥。
最后你应该看到类似的东西：
`[  220.784509] cloud-init[1358]: Cloud-init v. 0.7.8 finished at ...`

## 6、登录Ubuntu-cloud
现在在另一个终端中，你可以登录新启动的云服务器：
`ssh -p 2222 root@localhost`
如果一切顺利，你将直接登录，无需任何用户名或密码。
如果你以类似的方式启动了以前的QEMU图像，那么ssh可能会发出类似的严重警告（并拒绝登录）：
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
```
要解决此问题并删除以前的详细信息
`ssh-keygen -f ~/.ssh/known_hosts -R '[localhost]:2222'`
登录云服务器后，你可以确认它是一个aarch64系统
`uname -a`

## 7、卸载cloud-init
因为我们只是仿真来运行aarch64，所以并不需要cloud-int 卸载它
`apt remove -y cloud-init`

## 8、设置root密码并启用SSH的root登录
```
passwd root
#这里会要求你两遍输入一个默认密码
#请继续
vim /etc/ssh/sshd_config
#找到并用#注释掉这行：PermitRootLogin prohibit-password
#新建一行 添加：PermitRootLogin yes
#重启ssh服务

service ssh restart

#现在你可以尝试使用另外的一台机器
#或者不同的用户来登录Ubuntu-cloud
```

## 9、修改启动命令行，重启镜像
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
    -device virtio-net-device,netdev=user0 \
    -netdev user,id=user0 \
    -redir tcp:2222::22
```
如果一切正常，那么恭喜你，你在X86机器上仿真运行了aarch64的云镜像！
同样是结合之前的文档和记忆写的，所以较乱，请多包涵
如果有什么问题或者不足之处，请留言给我哦！
[进阶操作请点击查看我的下一篇文章](https://blog.lpxin.com/2019/05/16/QEMU%E8%BF%9B%E9%98%B6%E6%93%8D%E4%BD%9C-%E4%BD%BF%E7%94%A8%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F-%E6%89%A9%E5%AE%B9%E9%95%9C%E5%83%8F-%E5%B9%B6%E5%88%9B%E5%BB%BA%E5%A2%9E%E9%87%8F%E9%95%9C%E5%83%8F/ "QEMU进阶操作-使用桥接模式-扩容镜像-并创建增量镜像")

## 10、对于想要在arm架构内运行QEMU的可以使用KVM
KVM我这里不多做介绍了，只需要知道KVM模式下可以得到和宿主机一样的性能，且资源占用较少。
现在市面上大部分的开发板都支持KVM模式了，我这里以N1为例来演示。
其他步骤都和上面是一样的，只需要-cpu部分稍作修改即可,我直接放出修改后的启动命令行。
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
    -device virtio-net-device,netdev=user0 \
    -netdev user,id=user0 \
    -redir tcp:2222::22
```
如果你是希望通过QEMU来实现N1多挂博纳云，[那么你还需要继续点击阅读QEMU进阶操作](https://blog.lpxin.com/2019/05/16/QEMU%E8%BF%9B%E9%98%B6%E6%93%8D%E4%BD%9C-%E4%BD%BF%E7%94%A8%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F-%E6%89%A9%E5%AE%B9%E9%95%9C%E5%83%8F-%E5%B9%B6%E5%88%9B%E5%BB%BA%E5%A2%9E%E9%87%8F%E9%95%9C%E5%83%8F/ "QEMU进阶操作")

**end**
