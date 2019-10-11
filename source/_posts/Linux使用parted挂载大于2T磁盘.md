---
title: Linux使用parted挂载大于2T磁盘
date: 2019-05-16 13:55:10
tags: [Linux,parted,fdisk,Ubuntu]
categories: Linux
---
<center>
介绍2种分区表：<br/>
MBR分区表：（MBR含义：主引导记录）<br/>
所支持的最大卷：2T （T; terabytes,1TB=1024GB）<br/>
对分区的设限：最多4个主分区或3个主分区加一个扩展分区。<br/>
GPT分区表：（GPT含义：GUID分区表）<br/>
支持最大卷：18EB，（E：exabytes,1EB=1024TB）<br/>
每个磁盘最多支持128个分区<br/>
而遗憾的是我们常用的fdisk工具不支持GPT<br/>
得使用另一个GNU发布的强大分区工具parted
</center>

![遗憾](/images/post/linux-parted.jpeg "遗憾")
<!--more-->

## fdisk尝试操作
首先我们先对一个大于2T的磁盘进行操作看看他的报错
```
# fdisk -l

Disk /dev/sdb: 3 TiB, 3221225472000 bytes, 6291456000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

#可以看到我这里大于2T的磁盘为/dev/sdb

# fdisk /dev/sdb

Device does not contain a recognized partition table.
The size of this disk is 3 TiB (3221225472000 bytes). DOS partition table format cannot be used on drives for volumes larger than 2199023255040 bytes for 512-byte sectors. Use GUID partition table format (GPT).

#可以看到报错信息，并要求我们使用使用GUID分区表格式（GPT）。
#其实这里我们可以正常进行分区创建操作的，但是最大只能创建2T空间的分区，因为分区表格式为mbr。
#所以我就不进行操作了，有能力的可以自己尝试，这很简单。

```

## 使用parted创建分区
```
# parted /dev/sdb
#这里换为你自己需要分区的磁盘设备名

GNU Parted 3.2
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) p  #查看分区状态
Error: /dev/sdb: unrecognised disk label
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 3221GB
#请记住这里的大小，如果你想使用全部的磁盘空间的话，这个容量将会在后面使用到
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags:
(parted) mklabel gpt
#创建磁盘标签为gpt
#如果你的磁盘里有数据的话，这里会出现警告，继续操作将会销毁所有数据
#如果你的数据都已经备份的话，请根据提示键入y并回车
(parted) mkpart
Partition name?  []? sdb1  #指定分区名称
File system type?  [ext2]? ext4  #指定分区类型
Start? 0  #指定分区开始位置
End? 3221GB  #指定分区结束位置
Warning: The resulting partition is not properly aligned for best performance.
Ignore/Cancel? Ignore
(parted) p  #显示分区信息，可以看到分区已经完成，大小为3221GB
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 3221GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      17.4kB  3221GB  3221GB  ext4         sdb1

(parted) quit  #quit退出
Information: You may need to update /etc/fstab.
```
如果是fdisk进行分区操作的话，结束后还需要将其格式化为ext4
`mkfs.ext4 /dev/sdb`
parted就不需要进行这步操作，因为之前在分区时已经格式化为ext4了，
当然，你再格式化一次也没有问题。

## 挂载磁盘/dev/sdb
刚刚已经将磁盘/dev/sdb进行了分区操作，
我们得到了/dev/sdb1，现在我们将其挂载到用户主目录
```
# mkdir temp
# mount /dev/sdb1 temp

wrong fs type, bad option, bad superblock on /dev/sdb1, missing codepage or helper program, or other error.
```
不幸的是，我获得了一个报错。
我尝试用mkfs.ext4 /dev/sdb对其进行格式化后一切正常。
```
# mkfs.ext4 /dev/sdb1

mke2fs 1.44.1 (24-Mar-2018)
Creating filesystem with 786431991 4k blocks and 196608000 inodes
Filesystem UUID: 845e6f0c-27bb-40a2-884c-5f3cc2834a52
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done

# mount /dev/sdb1 temp
# df -h

Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           393M  1.5M  391M   1% /run
/dev/sda2        20G  5.7G   13G  31% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       91M   91M     0 100% /snap/core/6350
tmpfs           393M     0  393M   0% /run/user/1000
/dev/sdb1       2.9T   89M  2.8T   1% /home/anshan/temp
```
可以看到，刚刚我们分区操作的/dev/sdb1已经挂载到我的用户主目录下的temp上了。
如果想取消挂载，请尝试下面的命令
`umount temp`

## 设置开机自动挂载
```
echo "/dev/sdb1 /home/anshan/temp ext4 defaults 0 0" >> /etc/fstab

#设备名和路径请根据你的实际情况继续修改。
#请注意，这里有>>两个向左的箭头，如果只键入了一个箭头，那么很不辛，
#你的系统会出现异常，因为>>表示追加写入，而>是覆盖写入。
```
**end**
