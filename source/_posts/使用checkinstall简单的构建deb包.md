---
title: 使用checkinstall简单的构建deb包
date: 2019-05-21 09:09:36
tags: [Debian,Ubuntu,checkinstall,deb]
categories: Ubuntu
---
<center>
如果你有一个需要源码编译安装的软件需要共享给他人<br/>
或者你已经从它的源码运行make install安装了linux程序<br/>
想完整移除它将变得麻烦<br/>
除非程序的开发者在Makefile里提供了uninstall的目标设置<br/>
否则你必须在安装前后比较你系统里文件的完整列表<br/>
然后手工移除所有在安装过程中加入的文件<br/>
而Checkinstall将帮助你简单的实现卸载
</center>
<!--more-->

## Checkinstall
Checkinstall会跟踪install命令行所创建或修改的所有文件的路径(例如：”make install”、”make install_modules”等)并建立一个标准的二进制包，让你能用你发行版的标准包管理系统安装或卸载它，(例如Red Hat的yum或者Debian的apt-get命令)

而且Checkinstall不仅可以生成deb包, 还可以生成rpm包，使用简单，但是不灵活，功能粗糙，但是他适合从源代码直接构建我们的deb包, 我们下载到待打包的源代码以后, 先使用make进行编译, 然后运行checkinstall即可完成deb的打包,如果只是打包来给自己使用，那么它将会是一个不错的选择。

## 安装Checkinstall
我这里以Ubuntu18.04为例

`apt-get install checkinstall`

可以使用`checkinstall --help`来查看帮助信息

## 编译deb包
我这里以QEMU-3.1.0为例

```
wget https://download.qemu.org/qemu-3.1.0.tar.xz
tar xvJf qemu-3.1.0.tar.xz
cd qemu-3.1.0
apt update && apt install libglib2.0-dev libtool libpixman-1-dev flex bison python bridge-utils screen -y
./configure --target-list=aarch64-softmmu
make -j4


checkinstall -D --pkgname=qemu-3.1.0 --pkgversion=2019-05-21 --install=no  --pkgsource=../QEMU3.1-deb

--pkgname：deb包的软件名
--pkgversion： 打包时间
--install：打包完成后是否安装
--pkgsource：生成deb包的存放目录
```
运行checkinstall命令后他会给出一些提示信息，是否生成文档，输入包描述等，一般默认即可。

打包完成后，你将在上级的QEMU3.1-deb目录得到deb包。

**end**
