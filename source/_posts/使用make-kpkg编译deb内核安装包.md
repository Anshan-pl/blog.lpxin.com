---
title: 使用make-kpkg编译deb内核安装包
date: 2019-05-15 19:27:58
tags: [kernel,make-kpkg,fakeroot,Debian,Ubuntu]
categories: Ubuntu
---
<center>
上一篇文章介绍了交叉编译安装kernel<br/>
这一章使用make-kpkg交叉编译kernel安装包<br/>
这意味着你可以将生成的.deb文件传输到其他Debian\Ubuntu系统并以完全相同的方式安装它们<br/>
而不用在那里编译内核
</center>
<!--more-->

## 配置编译环境
### 1、之前的文章里已经介绍了如何安装aarch64-gcc
所以我们跳过这一步
### 2、安装make-kpkg：
```
apt-get install git build-essential fakeroot kernel-package u-boot-tools zlib1g-dev libncurses5-dev -y
```
### 3、设置环境变量
```
export ARCH = arm 
export DEB_HOST_ARCH = arm64
export CONCURRENCY_LEVEL =`grep -m1 cpu \ cores / proc / cpuinfo | cut -d：-f 2`
#或者export CONCURRENCY_LEVEL = 4进行编译
touch REPORTING-BUGS
#运行这个命令是因为之前在编译的过程中遇到了“cannot stat 'REPORTING-BUGS'：No such file or directory”的问题，通过google，使用上面的方法，即创建REPORTING-BUGS来进行解决

```
在进行下一步之前假设您已经具备了aarch64-gcc编译器以及需要的.config配置文件，如果没有
[请阅读这篇文章](https://blog.lpxin.com/2019/05/15/%E5%9C%A8Ubuntu%E4%B8%8A%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91arm%E5%BC%80%E5%8F%91%E6%9D%BF%E5%86%85%E6%A0%B8%E5%B9%B6%E5%AE%89%E8%A3%85)
### 4、编译你需要的内核
```
fakeroot make-kpkg --arch arm64 --cross-compile /root/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu- --initrd --append-to-version  -custom1 kernel_image kernel_headers

#编译过程中可能会出现一些warning dpkg-architecture: warning: specified GNU system type aarch64-linux-gnu does not match CC system type x86_64-linux-gnu, try setting a correct CC environment variable的警告，这是告诉你编译的内核和当前框架不匹配，忽略就行
```
### 5、安装编译完成的内核
编译之后，在内核源代码树上一级的目录中，您将获得：
```
linux-image-*.deb   ##一个带内核的Debian包;
linux-headers-*.deb  ###带有内核头文件的Debian软件包; 

```
将编译完成的两个deb复制到需要安装新内核的系统
`dpkg -i *.deb`
### 6、安装完成以后，重新启动系统，验证内核的版本
`uname -sr`
你现在可以检查/boot/grub/menu.lst，你应该在那里找到两个新内核节：
`cat /boot/grub/menu.lst`
