---
title: Linux基础命令总结
date: 2019-08-16 19:34:56
tags: [Linux,Shell]
categories: Linux
---
<center>
这篇文章里介绍的所有命令</br>
默认使用的终端是Bash(Bourne-Again SHell）解释器
</center>
<!--more-->
# 
在Linux系统中的命令参数有长短格式之分，长格式和长格式之间不能合并，长格式和短格式之间也不能合并，但短格式和短格式之间是可以合并的，合并后仅保留一个-（减号）即可。

# date
date命令中输入以“+”号开头的参数，即可按照指定格式来输出系统的时间或日期。


|参数|作用|
|:-:|:-:|
|%t|跳格[Tab键]|
|%H|小时（00～23）|
|%l|小时（00～23）|
|%M|分钟（00～59）|
|%S|秒（00～59）|
|%j|今年中的第几天|

```
# 按照“年-月-日 小时:分钟:秒”的格式查看当前系统时间的date命令如下所示：
date "+%Y-%m-%d %H:%M:%S"
2019-08-14 07:04:43

# 将系统的当前时间设置为2019年9月1日8点30分的date命令如下所示：
date -s "20190901 8:30:00"

# date命令中的参数%j可用来查看今天是当年中的第几天。这个参数能够很好地区分备份时间的新旧，即数字越大，越靠近当前时间。该参数的使用方式以及显示结果如下所示。
date "+%j"
```
# wget
wget命令的参数以及作用


|参数|作用|
|:-:|:-:|
|-b|后台下载模式|
|-P(大写P)|下载到指定目录|
|-t|最大尝试次数|
|-c|断点续传|
|-p(小写p)|下载页面内所有资源，包括图片、视频等|
|-r|递归下载|

# ps
ps命令用于查看系统中的进程状态，格式为“ps [参数]”。
Linux系统中时刻运行着许多进程，如果能够合理地管理它们，则可以优化系统的性能。在Linux系统中，有5种常见的进程状态，分别为运行、中断、不可中断、僵死与停止，其各自含义如下所示。


|进程状态|含义|
|:-:|:-:|
|R（运行）|进程正在运行或在运行队列中等待。|
|S（中断）|进程处于休眠中，当某个条件形成后或者接收到信号时，则脱离该状态。|
|D（不可中断）|进程不响应系统异步信号，即便用kill命令也不能将其中断。|
|Z（僵死）|进程已经终止，但进程描述符依然存在, 直到父进程调用wait4()系统函数后将进程释放。|
|T（停止）|进程收到停止信号后停止运行。|

常用ps命令会搭配aux或是ef参数，例如ps -aux或ps ef。
如前面所提到的，在Linux系统中的命令参数有长短格式之分，长格式和长格式之间不能合并，长格式和短格式之间也不能合并，但短格式和短格式之间是可以合并的，合并后仅保留一个-（减号）即可。另外ps命令可允许参数不加减号（-），因此可直接写成ps aux的样子。


|参数|作用|
|:-:|:-:|
|-a|显示所有进程（包括其他用户的进程）|
|-u|用户以及其他详细信息|
|-x|显示没有控制终端的进程|
|-e|显示所有进程|
|-f|全格式|

* ps -ef和ps aux，这两者的输出结果差别不大，但展示风格不同。aux是BSD风格，显示的项目有：USER , PID , %CPU , %MEM , VSZ , RSS , TTY , STAT , START , TIME , COMMAND。而-ef是System V风格，显示的项目有：UID , PID , PPID , C , STIME , TTY , TIME , CMD。
* COMMADN列如果过长，aux会截断显示，而ef不会。
* 综上，如果想查看进程的CPU占用率和内存占用率，可以使用aux ，如果想查看进程的父进程ID和完整的COMMAND命令，可以使用ef。

## 查看系统当前登陆用户命令
```
w
 22:29:43 up  1:42,  2 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     tty1     -                20:47   31:30   0.45s  0.45s -bash
anshan   pts/0    192.168.199.179  22:29    1.00s  0.02s  0.01s w



who
root     tty1         2019-08-26 20:47
anshan   pts/0        2019-08-26 22:29 (192.168.199.179)
```

## 查看自身用户名
```
whoami
anshan


who a mi
anshan   pts/0        2019-08-27 05:13 (192.168.199.179)
```
# LVM逻辑卷管理器
部署LVM时，需要逐个配置物理卷、卷组和逻辑卷。常用的部署命令如下所示：



|功能/命令|物理卷管理|卷组管理|逻辑卷管理|
|:-:|:-:|:-:|:-:|
|扫描|pvscan|vgscan|lvscan|
|建立|pvcreate|vgcreate|lvcreate|
|显示|pvdisplay|vgdisplay|lvdisplay|
|删除|pvremove|vgremove|lvremove|
|扩展||vgextend|lvextend|
|缩小||vgreduce|lvreduce|

## 使硬盘支持LVM
```
[root@localhost ~]# pvcreate /dev/sdb /dev/sdc
 Physical volume "/dev/sdb" successfully created
 Physical volume "/dev/sdc" successfully created
```

## 创建卷组
把两块硬盘设备加入到storage卷组中，然后查看卷组的状态，这里卷组的名称可以自定义修改。
```
[root@localhost ~]# vgcreate storage /dev/sdb /dev/sdc
 Volume group "storage" successfully created
[root@localhost ~]# vgdisplay
--- Volume group ---
 VG Name storage
 System ID 
 Format lvm2
 Metadata Areas 2
 Metadata Sequence No 1
 VG Access read/write
 VG Status resizable
 MAX LV 0
 Cur LV 0
 Open LV 0
 Max PV 0
 Cur PV 2
 Act PV 2
 VG Size 39.99 GiB
 PE Size 4.00 MiB
 Total PE 10238
 Alloc PE / Size 0 / 0  Free PE / Size 10238 / 39.99 GiB
 VG UUID KUeAMF-qMLh-XjQy-ArUo-LCQI-YF0o-pScxm1
………………省略部分输出信息………………
```
## 创建逻辑卷
从刚刚创建的storage卷组中使用5G来创建一个逻辑卷：
```
[root@localhost ~]# lvcreate -n lv0 -L 5G storage
WARNING: xfs signature detected on /dev/storage/lv0 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/storage/lv0.
  Logical volume "lv0" created.

#这里因为我之前已经进行过测试了，对其指定了xfs文件系统，所以这里会有个警官，键入y回车忽略即可。
```
-n：指定创建的逻辑卷名，默认的逻辑卷名为lvol*。
-L：指定创建的逻辑卷大小，这里指定的大小应小于等于卷组的大小。
-l：使用百分比来指定逻辑卷大小，如下例子：
```
[root@localhost ~]# lvcreate -n lv0 -l 50%VG storage
#创建一个使用卷组storage 50%空间大小的逻辑卷lv0

[root@localhost ~]# lvcreate -n lv0 -l 100%VG storage
#创建一个使用卷组storage 100%空间大小的逻辑卷lv0

[root@localhost ~]# lvcreate -n lv1 -l 50%FREE storage
#创建一个使用卷组storage剩余空间的50%的逻辑卷lv0
```
## 格式化逻辑卷并挂载
Linux系统会把LVM中的逻辑卷设备存放在/dev设备目录中（实际上是做了一个符号链接），同时会以卷组的名称来建立一个目录，其中保存了逻辑卷的设备映射文件（即/dev/卷组名称/逻辑卷名称）。
这里格式化逻辑卷使用了两种常见的文件系统xfs和ext4：
## 指定文件系统为xfs
```
[root@localhost ~]# mkfs.xfs /dev/storage/lv0
[root@localhost ~]# mount /root/temp /dev/storage/lv0
[root@localhost ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 969M     0  969M   0% /dev
tmpfs                    984M     0  984M   0% /dev/shm
tmpfs                    984M  8.8M  975M   1% /run
tmpfs                    984M     0  984M   0% /sys/fs/cgroup
/dev/mapper/rhel-root     37G  1.6G   36G   5% /
/dev/sda1               1014M  146M  869M  15% /boot
tmpfs                    197M     0  197M   0% /run/user/0
/dev/mapper/storage-lv0   10G  104M  9.9G   2% /root/temp

#通过df命令可以看到刚刚创建的卷组lv0已经被挂载到了/root/temp目录下，且容量为10G。
```
## 指定文件系统为ext4
```
[root@localhost ~]# mkfs.ext4 /dev/storage/lv0
```
剩下挂载步骤同上。

## 扩展逻辑卷容量
```
[root@localhost ~]# lvextend -L 15G /dev/storage/lv0
  Size of logical volume storage/lv0 changed from <10.00 GiB (2559 extents) to 15.00 GiB (3840 extents).
  Logical volume storage/lv0 successfully resized.
```

## 重置硬盘容量，在线生效——xfs
使用xfs_growfs命令可以使扩容立即生效，且是逻辑盘在线操作的，无需卸载：
```
[root@localhost ~]# xfs_growfs /root/temp
meta-data=/dev/mapper/storage-lv0 isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 2620416 to 3932160

#注意：这条命令后的参数是挂载的目录，而不是逻辑卷的路径。
```

## 重置硬盘容量，在线生效——ext4
使用resize2fs命令同样可以使扩容立即生效，且在线操作，无需卸载：
```
resize2fs /dev/storage/lv0
resize2fs 1.44.3 (10-July-2018)
Filesystem at /dev/storage/lv0 is mounted on /root/temp; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 2
The filesystem on /dev/storage/lv0 is now 3932160 (4k) blocks long.

#注意：这条命令后的参数是逻辑卷路径，不是挂载目录了。
```
# XFS文件系统修复及备份
任何修复都是有风险的，建议常备份业务数据！

## 判断故障类型
注意：如果是输入输出错误不一定是文件系统损坏，可以通过xfs_check命令判断：
```
[root@localhost ~]# xfs_ncheck /dev/storage/lv0
        131 temp/.
        132 1
        133 2
        134 3
        135 temp/5
        137 6
#这是正常状态下，该命令会返回分区内的文件信息，表示文件系统正常。

[root@localhost ~]# xfs_ncheck /dev/storage/lv0
Metadata CRC error detected at 0x55c7bf8b65a2, xfs_agi block 0x2/0x200
xfs_ncheck: cannot init perag data (74). Continuing anyway.
Metadata CRC error detected at 0x55c7bf8888f4, xfs_agfl block 0x3/0x200
Metadata CRC error detected at 0x55c7bf8c0751, xfs_refcountbt block 0x10/0x1000
bad magic # 0 in refcntbt block 0/2
Metadata CRC error detected at 0x55c7bf8bdd31, xfs_inobt block 0x0/0x1000
bad magic # 0x58465342 in inobt block 0/0
#有error信息说明了元数据错误，这时候就是文件系统损坏了。
```
如果不是文件系统损坏，可以简单的通过重新挂载来尝试解决问题，但这不是长久之计，很大可能是硬盘出现故障了。

## 破坏XFS
我这里直接使用了dd命令破坏了文件系统
```
[root@localhost ~]# dd if=/dev/zero of=/dev/storage/lv0 bs=1K count=1024
1024+0 records in
1024+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.00313658 s, 334 MB/s
```
可以使用hexdump -C -n命令对比前后变化。

## 备份XFS元数据
在执行了刚刚的dd命令故意覆盖第一兆字节的/dev/storage/lv0逻辑卷或者分区、硬盘后该分区下可能会报I/O error，或者一些其他异常现象，当尝试重新挂载发现已经挂载不上了，如下：
```
[root@localhost ~]# umount /root/temp/
[root@localhost ~]# mount /dev/storage/lv0 /root/temp
mount: /root/temp: mount(2) system call failed: Structure needs cleaning.
```
现在找一个临时目录来存放元数据，这里我使用的是/tmp/temp目录运行xfs_metadump命令来进行备份：
```
[root@localhost ~]# xfs_metadump /dev/storage/lv0 /tmp/temp/lv0.metadump
Metadata CRC error detected at 0x560353cd25c2, xfs_agf block 0x1/0x200
xfs_metadump: cannot init perag data (74). Continuing anyway.
Metadata CRC error detected at 0x560353d005a2, xfs_agi block 0x2/0x200
Metadata CRC error detected at 0x560353cd28f4, xfs_agfl block 0x3/0x200
```
现在/dev/storage/lv0中的元数据已经备份至lv0.metadump中了，接下来将元数据还原到图像中,以便可以执行修复并查看损坏情况：
```
[root@localhost ~]# xfs_mdrestore /tmp/temp/lv0.metadump /tmp/temp/lv0.img
[root@localhost ~]# mkdir /tmp/lv0
[root@localhost ~]# mount /tmp/temp/lv0.img /tmp/lv0
mount: /tmp/lv0: mount(2) system call failed: Structure needs cleaning.

#直接进行挂载测试是挂载不成功的，因为该XFS图像还未进行修复，接下来对该图像进行修复尝试：
[root@localhost ~]# xfs_repair /tmp/temp/lv0.img
………………省略部分输出信息………………
bad CRC for inode 191, will rewrite
bad magic number 0x0 on inode 191, resetting magic number
bad version number 0x0 on inode 191, resetting version number
inode identifier 0 mismatch on inode 191
cleared inode 191
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
root inode lost
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
reinitializing root directory
reinitializing realtime bitmap inode
reinitializing realtime summary inode
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
resetting inode 128 nlinks from 1 to 2
done

#已经修复成功了，挂载尝试：
[root@localhost ~]# mount /tmp/temp/lv0.img /tmp/lv0

```
修复后挂载已经挂载成功了，查看其中的数据发现有丢失，具体权衡要靠自己把握了，如果确定丢失部分在承受范围内可以直接对逻辑卷进行修复操作后重新挂载。

这里如果修复不成功的话还可以对其加上-L参数，该命令会擦除所有日志信息，会造成更多的数据丢失！
对于-L参数说明：-L是修复xfs文件系统的最后手段，慎重选择，它会清空日志，会丢失用户数据和文件。
