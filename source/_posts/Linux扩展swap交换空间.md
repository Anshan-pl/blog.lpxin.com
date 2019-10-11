---
title: Linux扩展swap交换空间
date: 2019-05-16 14:42:31
tags: [Linux,swap,dd,mkswap]
categories: Linux
---
<center>
SWAP就是LINUX下的虚拟内存分区<br/>
它的作用是在物理内存使用完之后<br/>
将磁盘空间(也就是SWAP分区)虚拟成内存来使用<br/>
linux可以文件或者分区来当作虚拟内存
</center>
<!--more-->

## 首先查看当前的内存和swap空间大小
```
# free -m

#默认单位为k, -m 单位为M, -h 单位为G

# swapon -s
#或者
# cat /proc/swaps
```
如果都没有，我们就需要手动添加交换分区。注意，OPENVZ架构的VPS是不支持手动添加交换分区的。 
添加交换空间有两种选择：添加一个交换分区或添加一个交换文件。
推荐你添加一个交换分区；不过，若你没有多少空闲空间可用， 则添加交换文件。

## 使用连续交换文件
### 创建swap交换文件
`dd if=/dev/zero of=/home/swap bs=1024 count=2048000`
这里的/dev/zero是用来产生一个特定大小的空白文件。
这样就建立一个/home/swap的分区文件，大小为2G。
你可以根据你的情况来修改交换文件的大小。

### 制作为swap格式的文件
`mkswap /home/swap`

### 将swap文件挂载为swap分区
`swapon /home/swap`

### 再次查看当前的内存和swap空间大小
命令同第一次，我们发现swap的大小已经扩大了2G

### 设置引导时自动启用自定义的swap文件
`echo "/home/swap swap swap default 0 0" >> /etc/fstab`

### swap命令
swapon -a  #开启全部交换分区
swapoff -a  #关闭全部交换分区，如果需要运行k8s则运行关闭全部swap
swapoff /home/swap  #关闭指定swap分区/文件

如果需要取消swap文件的开机自动挂载，只需要进入/etc/fstab将swap行注释或删除即可。

## 使用swap分区
提示：连续交换文件和交换分区之间没有性能之别，两者的处理方式是一样的。
[参考这里](https://wiki.archlinux.org/index.php/Swap_ "交换文件与交换分区")

### 设置swap分区
mkswap /dev/sdb1

### 启用swap分区
swapon /dev/sdb1

### 设置引导启用swap分区
`echo "/dev/sdb1 swap swap defaults 0 0" >> /etc/fstab`

### 删除swap分区
#### 停止swap分区
swapoff /dev/sdb1
#### 修改/etc/fstab
`#/dev/sdb1 swap swap defaults 0 0`
将自定义的swap分区行删除或注释即可。

如果有什么问题或者不对的地方，欢迎留言指正~
我会随时查看。

**end**
