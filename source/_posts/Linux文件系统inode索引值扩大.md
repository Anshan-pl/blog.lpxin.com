---
title: Linux文件系统inode索引值扩大
date: 2019-10-19 16:09:50
tags: [Linux,ext,ext4,xfs,inode,mkfs]
categories: inode
---

<center>
在linux生产环境中遇到了硬盘还有容量
但无法写入的问题，提示为：
No space left on device
造成该问题的原因为硬盘inode已满
</center>
<!--more-->

# inode问题复现
这里我使用的是虚拟机复现该问题：
```
[root@localhost test]# echo "hello_world" > /root/test/index
-bash: /root/test/index: No space left on device
#可以看到，这里在向/root/test目录中写入数据时提示空间不足。
#但是，我们继续查看下空间容量：

[root@localhost test]# df -Th
Filesystem            Type      Size  Used Avail Use% Mounted on
devtmpfs              devtmpfs  898M     0  898M   0% /dev
tmpfs                 tmpfs     910M     0  910M   0% /dev/shm
tmpfs                 tmpfs     910M  9.6M  900M   2% /run
tmpfs                 tmpfs     910M     0  910M   0% /sys/fs/cgroup
/dev/mapper/rhel-root xfs        37G  1.6G   36G   5% /
/dev/sda2             xfs      1014M  172M  843M  17% /boot
/dev/sda1             vfat      200M   10M  190M   5% /boot/efi
tmpfs                 tmpfs     182M     0  182M   0% /run/user/0
/dev/sdb              ext4      8.7M  220K  7.8M   3% /root/test
#这里可以看到，/root/test目录下还有很多容量，但是向其写入数据却提示空间不足。
#接下来继续查看/root/test挂载硬盘/dev/sdb的文件索引剩余值：

[root@localhost ~]# df -Ti
Filesystem            Type       Inodes IUsed    IFree IUse% Mounted on
devtmpfs              devtmpfs   229790   419   229371    1% /dev
tmpfs                 tmpfs      232807     1   232806    1% /dev/shm
tmpfs                 tmpfs      232807   786   232021    1% /run
tmpfs                 tmpfs      232807    16   232791    1% /sys/fs/cgroup
/dev/mapper/rhel-root xfs      19292160 39420 19252740    1% /
/dev/sda2             xfs        524288    23   524265    1% /boot
/dev/sda1             vfat            0     0        0     - /boot/efi
tmpfs                 tmpfs      232807     1   232806    1% /run/user/0
/dev/sdb              ext4         2560  2560        0  100% /root/test
#使用df -Ti看到/dev/sdb盘的可使用inode值已经为0，如果遇到这种情况，唯一可作的，就是扩展该盘的inode最大值。
```
# 格式化文件系统时指定bytes-per-inode大小
这里inode size值决定了文件系统的inode数量，因为硬盘的容量是一定的，而inode最大值由硬盘容量和inode size决定，文件索引节点的单位为字节。
```
[root@localhost ~]# mkfs.ext4 /dev/sdb1 -i 8192
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
13107200 inodes, 26214144 blocks
1310707 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2174746624
800 block groups
32768 blocks per group, 32768 fragments per group
16384 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

#这里inode索引字节范围如下：
[root@localhost ~]# mkfs.ext4 -i 2
mkfs.ext4: invalid inode ratio 2 (min 1024/max 67108864)
```
可以看到这里指定inode size为8196，理论上为inode size值越小，inode数量越多，值越大。


# 单硬盘扩容，实现inode扩大
这里，因为我使用的是虚拟机，扩容实现比较方便，给出扩容方法。
扩容前，先使用df -i查看硬盘inode值：
```
[root@localhost ~]# df -i
Filesystem              Inodes IUsed    IFree IUse% Mounted on
devtmpfs                229790   422   229368    1% /dev
tmpfs                   232807     1   232806    1% /dev/shm
tmpfs                   232807   792   232015    1% /run
tmpfs                   232807    16   232791    1% /sys/fs/cgroup
/dev/mapper/rhel-root 19292160 39437 19252723    1% /
/dev/sda2               524288    23   524265    1% /boot
/dev/sda1                    0     0        0     - /boot/efi
tmpfs                   232807     1   232806    1% /run/user/0
/dev/sdb                  2304    11     2293    1% /root/test
```
扩容前inode值为2304，下面对/dev/sdb进行扩容操作：
```
[root@localhost ~]# e2fsck -f /dev/sdb
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/sdb: 2560/2560 files (0.1% non-contiguous), 1573/10240 blocks

[root@localhost ~]# resize2fs /dev/sdb
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/sdb to 524288 (1k) blocks.
The filesystem on /dev/sdb is now 524288 blocks long.

[root@localhost ~]# mount /dev/sdb /root/test

[root@localhost ~]# df -i
Filesystem              Inodes IUsed    IFree IUse% Mounted on
devtmpfs                229790   419   229371    1% /dev
tmpfs                   232807     1   232806    1% /dev/shm
tmpfs                   232807   786   232021    1% /run
tmpfs                   232807    16   232791    1% /sys/fs/cgroup
/dev/mapper/rhel-root 19292160 39422 19252738    1% /
/dev/sda2               524288    23   524265    1% /boot
/dev/sda1                    0     0        0     - /boot/efi
tmpfs                   232807     1   232806    1% /run/user/0
/dev/sdb                 81920  2560    79360    4% /root/test
```
可以看到， 扩容后文件索引数inode值已经扩展为81920，且不影响该盘数据。

# 分区扩容，实现inode扩大
分区扩容不同于硬盘直接扩容，需要先使用fdisk工具对分区进行扩容操作。
扩容前， 先查看下分区的inode值：
```
[root@localhost ~]# df -i
Filesystem              Inodes IUsed    IFree IUse% Mounted on
devtmpfs                229790   422   229368    1% /dev
tmpfs                   232807     1   232806    1% /dev/shm
tmpfs                   232807   792   232015    1% /run
tmpfs                   232807    16   232791    1% /sys/fs/cgroup
/dev/mapper/rhel-root 19292160 39437 19252723    1% /
/dev/sda2               524288    23   524265    1% /boot
/dev/sda1                    0     0        0     - /boot/efi
tmpfs                   232807     1   232806    1% /run/user/0
/dev/sdb1                 2304    11     2293    1% /root/test
```
扩容前为2304，接下来使用fdisk工具对分区进行扩容：
```
[root@localhost ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p

Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xda0c88d3

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048       20479        9216   83  Linux

Command (m for help): d
Selected partition 1
Partition 1 is deleted

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p):
Using default response p
Partition number (1-4, default 1):
First sector (2048-209715199, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-209715199, default 209715199):    #注意：这里因为我是测试，所有直接使用了默认最大值，将全部容量分配给这个主分区，如果你还需要预留空间，那么请指定扇区。
Using default value 209715199
Partition 1 of type Linux and of size 100 GiB is set

Command (m for help): p

Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xda0c88d3

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   209715199   104856576   83  Linux

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```
现在已经对该分区进行了扩容操作，重新对其格式化后即可看到变化，如果不想重新格式化而保留原分区数据，那么可以使用resize2fs命令来同步分区表大小：
```
[root@localhost ~]# resize2fs /dev/sdb1
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/sdb1 is mounted on /root/test; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 800
The filesystem on /dev/sdb1 is now 104856576 blocks long.

#重新检查sdb1分区的inode值：
[root@localhost ~]# df -i
Filesystem              Inodes IUsed    IFree IUse% Mounted on
devtmpfs                229790   422   229368    1% /dev
tmpfs                   232807     1   232806    1% /dev/shm
tmpfs                   232807   792   232015    1% /run
tmpfs                   232807    16   232791    1% /sys/fs/cgroup
/dev/mapper/rhel-root 19292160 39436 19252724    1% /
/dev/sda2               524288    23   524265    1% /boot
/dev/sda1                    0     0        0     - /boot/efi
tmpfs                   232807     1   232806    1% /run/user/0
/dev/sdb1             14745600    12 14745588    1% /root/test
```
可以看到修改已经生效，且不影响原盘数据。

# 对LVM逻辑卷进行扩容，实现inode扩大
在写文章之前我自己进行了测试，所以这里我先将扩容LVM添加的硬盘给删除：
```
#先从lv中将扩容的容量缩减
[root@localhost ~]# lvreduce -L -100G /dev/mapper/rhel-root
  WARNING: Reducing active and open logical volume to <36.80 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce rhel/root? [y/n]: y
  Size of logical volume rhel/root changed from <136.80 GiB (35020 extents) to <36.80 GiB (9420 extents).
  Logical volume rhel/root successfully resized.

#接下来从vg中删除扩容的PV物理盘
[root@localhost ~]# vgreduce rhel /dev/sdb
  Removed "/dev/sdb" from volume group "rhel"

#继续从pv中移除硬盘
[root@localhost ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.

#使用xfs_growfs命令进行调整
[root@localhost ~]# xfs_growfs /dev/mapper/rhel-root
meta-data=/dev/mapper/rhel-root  isize=512    agcount=15, agsize=2411520 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=35860480, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=4710, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data size 9646080 too small, old size is 35860480
```
好了， 现在/dev/sdb盘已经是一块空闲盘了，接下来将用sdb盘对根目录进行扩容。
扩容前先看下根目录的inode值：

**注意：这里，出现一个问题，就是我将xfs根目录缩容后系统起不来了，经过各种尝试后发现，xfs文件系统不支持缩容，请注意！**

```
[root@localhost ~]# lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root rhel -wi-ao---- <36.80g
  swap rhel -wi-ao----   2.00g

[root@localhost ~]# df -Ti
Filesystem            Type       Inodes IUsed    IFree IUse% Mounted on
devtmpfs              devtmpfs   250400   397   250003    1% /dev
tmpfs                 tmpfs      253417     1   253416    1% /dev/shm
tmpfs                 tmpfs      253417   758   252659    1% /run
tmpfs                 tmpfs      253417    16   253401    1% /sys/fs/cgroup
/dev/mapper/rhel-root xfs      19292160 37142 19255018    1% /
/dev/sda2             xfs        524288    22   524266    1% /boot
/dev/sda1             vfat            0     0        0     - /boot/efi
tmpfs                 tmpfs      253417     1   253416    1% /run/user/0
```
扩容前根目录36.8G，inode值19292160，下面我们进行扩容操作，先添加创建一个pv物理卷
```
[root@localhost ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@localhost ~]# pvs
  PV         VG   Fmt  Attr PSize  PFree
  /dev/sda3  rhel lvm2 a--  38.80g  4.00m
  /dev/sdb        lvm2 ---  20.00g 20.00g
```
创建pv支持硬盘或分区，这里Attr部分不一样，因为新创建的PV卷组还没有激活，a---表示已激活的pv卷组。VG那一栏是所属的VG组，而我还没有划分VG组，所以这一栏也是空。

继续像根目录的vg组中添加刚刚创建的盘，创建新vg组则使用vgcreate命令：
```
[root@localhost ~]# vgextend rhel /dev/sdb
  Volume group "rhel" successfully extended
[root@localhost ~]# vgs
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   2   2   0 wz--n- <58.80g 20.00g
```
同样，这里是扩容根目录，而不是创建，所以使用lvextend，如果是创建一个新的lv逻辑卷则使用lvcreate命令，两条命令的参数基本相同：
```
[root@localhost ~]# lvextend -L +20G /dev/mapper/rhel-root
  Size of logical volume rhel/root changed from <36.80 GiB (9420 extents) to <56.80 GiB (14540 extents).
  Logical volume rhel/root successfully resized.
[root@localhost ~]# lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root rhel -wi-ao---- <56.80g
  swap rhel -wi-ao----   2.00g
```
需要注意以下，扩容lv逻辑卷组是指定到具体的物理路径。可以看到， 扩容已经成功了，这里我最先使用的是-l 100%FREE参数使用vg的全部空间，但是不知道为什么没有成功，所以又改为-L参数手动指定扩容大小为+20G了，如下：
```
[root@localhost ~]# lvextend -l 100%FREE /dev/mapper/rhel-root
  New size given (5120 extents) not larger than existing size (9420 extents)
```
这个使用-l 100%FREE参数不成功的问题等下次再测试吧，下面继续调整分区容量并查看inode值是否增加：
```
[root@localhost ~]# xfs_growfs /dev/mapper/rhel-root
meta-data=/dev/mapper/rhel-root  isize=512    agcount=4, agsize=2411520 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=9646080, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=4710, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 9646080 to 14888960
[root@localhost ~]# df -Ti
Filesystem            Type       Inodes IUsed    IFree IUse% Mounted on
devtmpfs              devtmpfs   250400   412   249988    1% /dev
tmpfs                 tmpfs      253417     1   253416    1% /dev/shm
tmpfs                 tmpfs      253417   767   252650    1% /run
tmpfs                 tmpfs      253417    16   253401    1% /sys/fs/cgroup
/dev/mapper/rhel-root xfs      29777920 37128 29740792    1% /
/dev/sda2             xfs        524288    22   524266    1% /boot
/dev/sda1             vfat            0     0        0     - /boot/efi
tmpfs                 tmpfs      253417     1   253416    1% /run/user/0
```
扩容结束，inode值增加到了29777920。

# 删除空文件释放inode
使用find命令查找指定目录下所有0字符文件，结合管道符删除所有找到的空文件。
find . -name "*" -type f -size 0c|xargs -n 1 rm -f
```
[root@localhost ~]# find /root/test -name "*" -type f -size 0c|xargs -n 1 rm -f
#我这里是删除/root/test目录下的所有空文件，使用df -i命令查看现在的inode占用

[root@localhost ~]# df -Ti
Filesystem            Type       Inodes IUsed    IFree IUse% Mounted on
devtmpfs              devtmpfs   229790   419   229371    1% /dev
tmpfs                 tmpfs      232807     1   232806    1% /dev/shm
tmpfs                 tmpfs      232807   786   232021    1% /run
tmpfs                 tmpfs      232807    16   232791    1% /sys/fs/cgroup
/dev/mapper/rhel-root xfs      19292160 39422 19252738    1% /
/dev/sda2             xfs        524288    23   524265    1% /boot
/dev/sda1             vfat            0     0        0     - /boot/efi
tmpfs                 tmpfs      232807     1   232806    1% /run/user/0
/dev/sdb              ext4        81920    11    81909    1% /root/test
```
修改这里的.替换为你需要删除空文件的目录。

# xfs文件系统增加inode
xfs文件系统增加inode值与ext*文件系统方法是不同的，xfs可以直接动态扩大inode值，而不必对其进行扩容操作！
虚拟机里我添加了一个1G的硬盘，创建一个新的xfs文件系统来查看下inode值：
```
[root@localhost ~]# mkfs.xfs /dev/sdc
meta-data=/dev/sdc               isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@localhost ~]# mkdir test
[root@localhost ~]# mount /dev/sdc /root/test
[root@localhost ~]# df -Ti
Filesystem            Type       Inodes IUsed    IFree IUse% Mounted on
devtmpfs              devtmpfs   250400   420   249980    1% /dev
tmpfs                 tmpfs      253417     1   253416    1% /dev/shm
tmpfs                 tmpfs      253417   777   252640    1% /run
tmpfs                 tmpfs      253417    16   253401    1% /sys/fs/cgroup
/dev/mapper/rhel-root xfs      29777920 37131 29740789    1% /
/dev/sda2             xfs        524288    22   524266    1% /boot
/dev/sda1             vfat            0     0        0     - /boot/efi
tmpfs                 tmpfs      253417     1   253416    1% /run/user/0
/dev/sdc              xfs        524288     3   524285    1% /root/test
```
1G盘的inode值为29777920，下面增大inode值，首先使用xfs_info查看该xfs文件系统信息：
```
[root@localhost ~]# xfs_info /dev/sdc
meta-data=/dev/sdc               isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
这里的imaxpct=25就是inode值可用百分比，默认是给用户25%的inode值，将其调整为50%：
```
[root@localhost ~]# xfs_growfs
Usage: xfs_growfs [options] mountpoint

Options:
        -d          grow data/metadata section
        -l          grow log section
        -r          grow realtime section
        -n          don't change anything, just show geometry
        -i          convert log from external to internal format
        -t          alternate location for mount table (/etc/mtab)
        -x          convert log from internal to external format
        -D size     grow data/metadata section to size blks
        -L size     grow/shrink log section to size blks
        -R size     grow realtime section to size blks
        -e size     set realtime extent size to size blks
        -m imaxpct  set inode max percent to imaxpct
        -V          print version information

#可以看到，使用-m参数可以指定inode可用的百分比

[root@localhost ~]# xfs_growfs -m 50 /dev/sdc
meta-data=/dev/sdc               isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
inode max percent changed from 25 to 50
#已经提示inode max percent从25调整到50了，使用xfs_info查看下：

[root@localhost ~]# xfs_info /dev/sdc
meta-data=/dev/sdc               isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=262144, imaxpct=50
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
已经调整完成了，继续使用df -iT查看下现在的inode值：
```
[root@localhost ~]# df -iT
Filesystem            Type       Inodes IUsed    IFree IUse% Mounted on
devtmpfs              devtmpfs   250400   420   249980    1% /dev
tmpfs                 tmpfs      253417     1   253416    1% /dev/shm
tmpfs                 tmpfs      253417   777   252640    1% /run
tmpfs                 tmpfs      253417    16   253401    1% /sys/fs/cgroup
/dev/mapper/rhel-root xfs      29777920 37131 29740789    1% /
/dev/sda2             xfs        524288    22   524266    1% /boot
/dev/sda1             vfat            0     0        0     - /boot/efi
tmpfs                 tmpfs      253417     1   253416    1% /run/user/0
/dev/sdc              xfs       1048576     3  1048573    1% /root/test
```
/dev/sdc的inode值扩大为1048576，xfs调整inode值是可以在线动态调整的，无需停机或其他操作。
