---
title: 在Ubuntu上交叉编译arm开发板内核并安装
date: 2019-05-15 17:44:07
tags: [Ubuntu,kernel,armv8,aarch64,arm]
categories: Ubuntu
---
<center>
交叉编译开发板内核的原因，我想已经不用多说了...<br/>
用板子来编译安装kernel，我只能用四个字来形容，奇慢无比，无法承受...
</center>

![无法承受](/images/post/arm-kernel.jpg "无法承受")
<!--more-->

## 将开发板镜像挂载到硬盘
### 我这里以phicomm N1开发板，armbian镜像为例，来交叉编译4.19内核
```
losetup -P -f --show Armbian_5.76_Aml-s905_Ubuntu_bionic_default_4.20.5_20190224.img
#loop1为上面挂载的loop设备
#如果你是第一次使用losetup命令，那么这里你挂载的设备名也为loop1。
mount /dev/loop1p2 /mnt/
mount /dev/loop1p1 /mnt/boot/

```

## 安装编译工具并配置编译环境
### 1、配置编译环境，首先下载必要的工具
```
apt-get install gcc make pkg-config git bison flex libelf-dev libssl-dev libncurses5-dev -y
wget https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz
#如果你运行命令提示404 not found
#那么请自行到Linaro上查找最新版本（我这里例示的是2019年02月版本）
tar -Jxvf gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz
export ARCH=arm64
export CROSS_COMPILE=/root/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
#注意，这里你的路径可能与我的不同，请根据你的路径设置aarch64-gcc的环境变量
git clone https://github.com/150balbes/Amlogic_s905-kernel.git
cd Amlogic_s905-kernel
git checkout bc90907  #使用4.19.6的kernel

```
### 2、使用armbian自带内核的配置文件：
`cp /mnt/boot/config-4.20.5-aml-s905 .config`
### 3、配置要开启的功能，退出时要保存配置，更多编译选项使用make help查看。
```
make clean  #清理上一次编译的中间件
make menuconfig  #如果你不知道你在干什么，请逐次选择load-save-exit
```
### 4、编译安装内核文件：
```
make LOCALVERSION="-aml-s9xxx" Image -j 4 #这里的LOCALVERSION参数值你可以自行设置
#接下来是漫长的等待，视你的机器配置而定，可以悠闲的喝杯茶
make install INSTALL_PATH=/mnt/boot/
cp arch/arm64/boot/Image /mnt/boot/zImage
```
### 5、编译安装内核模块：
```
rm -rf /mnt/lib/modules
make LOCALVERSION="-aml-s9xxx" modules -j 4
#不必怀疑，这里会比刚刚的编译过程还要漫长
make modules_install INSTALL_MOD_PATH=/mnt/
```
### 6、安装至U盘
使用dd命令，或者在windows下使用usb_image_tool工具烧录刚刚修改好的镜像
