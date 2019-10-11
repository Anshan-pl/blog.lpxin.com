---
title: rhel重置root密码和修复引导
date: 2019-08-15 17:06:51
tags: [Redhat,rhel,root,grub,boot]
categories: Redhat
---

<center>
重置root密码需重启系统<br/>
修复引导需原系统的ISO映像<br/>
包括UEFI以及BIOS引导问题修复<br/>
以及fstab文件恢复
</center>
<!--more-->

# 重置root密码
1. 重启系统，在出现GRUB 菜单时按↑↓箭头或ESC键取消倒计时，之后将进入系统选择界面，也有可能直接进入操作系统选择项，视情况而定。
2. 在选择系统的界面按“e”进入编辑页面
3. 然后按向下键，找到以“Linux”开头的行，在该行的最后面输入“init=/bin/sh”,如下图所示。
![rehel_reset](/images/post/rhel_reset.png "rehel")
4. 接下来按“ctrl+X”组合键进入单用户模式
5. 接下来再输入“mount -o remount,rw /”(注意mount与－o之间和rw与/之间的有空格),这一步用于重新挂载根目录为可读写，否则为只读状态我们是无法修改密码的，因为无法写入到/etc/passwd和/etc/shadow文件。
6. 然后再输入“passwd root”回车，用于修改root密码，如果想修改其他用户的密码也是一样。
7. 接下来就是修改你的root账号密码了，重复输入一个不少于8位的密码，当然小于八位也可以，只不过系统会给你一个警告，忽略即可。（密码在输入的时候是不显示的，看起来就像没反应一样，只需要正确输入并回车就可以）
8. 输入touch /.autorelabel回车
9. 输入exec /sbin/init,回车
10. 密码重置完成，可以尝试使用新的密码进入系统了.

# GRUB修复，误删/boot/grub2
## 报错信息
```
error: ../../grub-core/fs/fshelp.c:258:file '/grub2/i386-pc/normal.mod' not found
Entering rescue mode....
grub rescue>
```

## 准备一个原版镜像光盘或者ISO文件并挂载
我这里是使用的rhel8测试版本系统，所以修复用的也是对应版本的ISO文件。
服务器物理机可以使用IPMI下的KVM挂载ISO文件并设置为第一启动项。
重新启动计算机后，当BIOS执行POST测试时，按特殊键（Esc，F2，F11，F12，Del，具体取决于主板说明），以进入BIOS设置并修改启动顺序，以便首先启动可启动DVD/USB映像在机器启动时。
VMware是在菜单栏：虚拟机-电源-打开电源时进入固件
如下图所示。
![rehel_post](/images/post/rhel_post.png "rehel")
将CD-ROM或USB启动项改为第一项后根据BIOS提供的保存热键保存并退出BIOS。

## 进入安装镜像中的故障排除项
在之后的Redhat8可启动媒体被检测到，安装选择界面将出现在您的屏幕输出。
从第一个菜单中选择故障排除（Troubleshooting）选项，然后按[回车]键继续。
![rehel_post2](/images/post/rhel_post2.png "rehel")

## 进入救援系统
接下来进入了一个子菜单，请选择第二项救援系统（Rescue a Red Hat Enterprise Linux system）并回车继续。
![rehel_post3](/images/post/rhel_post3.png "rehel")

## 继续系统恢复
安装程序软件加载到机器RAM后，救援环境提示将出现在您的屏幕上。在此提示符下键入1以继续系统恢复过程，如下图所示。
![rehel_post4](/images/post/rhel_post4.png "rehel")

## 修改救援系统下的层次结构
在下一个提示中，救援程序将通知您系统已安装在/mnt/sysimage目录下。这里，正如救援程序所建议的那样，键入chroot /mnt/sysimage，以便将Linux树层次结构从ISO映像更改为磁盘下安装的根分区。
```
首先键入回车键，获得一个shell
chroot /mnt/sysimage
```
## 查看硬盘名称
接下来，通过在救援提示中发出以下命令来识别您的机器硬盘驱动器。
```
ls /dev/sd*
```
如果您的计算机使用基础旧物理RAID控制器，则磁盘将具有其他名称，例如/dev/cciss。此外，如果您的rhel系统安装在虚拟机下，则可以将硬盘命名为/dev/vda或/dev/xvda等，一般的SATA、SAS盘都会被系统识别为sd*。

## 安装GRUB引导加载程序
在确定机器硬盘后，可以通过发出以下命令开始安装GRUB引导加载程序。
```
mkdir /boot/grub2
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-install /dev/sda
# 报错请忽略
exit
init 6
```
重启后进入系统会进行一系列恢复步骤，耐心等待即可。如果重启后还是进入了安装镜像，那么请在BIOS中修改回从硬盘引导，或者在Troubleshooting中选择Boot from local drive启动本地系统。

# BOOT目录修复，误删/boot
误删/boot目录下所有文件和误删/boot/grub2目录下所有文件的开机错误类似，修复方式也大致一样。

## 报错信息
```
Booting from local disk...
.
error: ../../grub-core/fs/fshelp.c:258:file '/grub2/i386-pc/normal.mod' not found.
Entering rescue mode...
grub rescue> 
```
## 进入救援模式并获取shell
步骤和上面的GRUB修复一样，进入原装镜像中的救援系统后按回车获取shell
修改救援系统的层次结构
```
chroot /mnt/sysimage
```
查看硬盘名称，使用ls /dev/sd*命令，方法也同上述的GRUB修复。

## 重新安装内核配置文件和引导程序
```
mount /dev/sr0 /mnt
# 挂载原装镜像到/mnt目录下，告警请忽略

ls /mnt/BaseOS/Packages/kernel*
# 查找内核文件，因为我这里安装的Redhat8测试版，所以Packages目录在BaseOS路径下，正常情况如下：
ls /mnt/Packages/kernel*

# 请根据自己的ls命令回显选择对应的内核文件安装包，我这里Redhat8测试版内核为4.18
rpm -ivh /mnt/BaseOS/Packages/kernel-4.18.0-80.e18.x86_64.rpm --force

ls /boot
# 应输出有文件信息

grub2-install /dev/sda
# 具体的硬盘名请根据自己的配置决定
grub2-mkconfig -o /boot/grub2/grub.cfg
exit
init 6
```
之后的步骤也同GRUB修复，引导入本地系统即可。

# 识别不到本地系统盘（引导记录损坏）
包含MBR引导代码的扇区称为主引导扇区。因这一扇区中，引导代码占有绝大部分的空间，故而将习惯将该扇区称为MBR扇区（简称MBR）。由于这一扇区承担有不同于磁盘上其他普通存储空间的特殊管理职能，作为管理整个磁盘空间的一个特殊空间，它不属于磁盘上的任何分区，所以分区空间内的格式化命令不能清除主引导记录的任何信息。
主引导扇区由三个部分组成(共占用512个字节)：
1. 主引导程序即主引导记录（MBR）（占446个字节）
	可用fdisk命令查看，它用于硬盘启动时将系统控制转给用户指定的并在分区表中登记了的某个操作系统。
2. 磁盘分区表项（DPT，Disk Partition Table)
	由四个分区表项构成（每个16个字节），负责说明磁盘上的分区情况，其内容由磁盘介质及用户在使用FDISK定义分区时决定。
3. 结束标志（占2个字节）
	其值为AA55，存储时低位在前，高位在后，即看上去是55AA（十六进制）。


对于硬盘上的MBR引导记录可以用hexdump命令查看：
`hexdump -C -n 512 /dev/sda`


## 模拟引导记录损坏
原硬盘前512字节信息如下：
```
hexdump -C -n 512 /dev/sda


00000000  eb 63 90 10 8e d0 bc 00  b0 b8 00 00 8e d8 8e c0  |.c..............|
00000010  fb be 00 7c bf 00 06 b9  00 02 f3 a4 ea 21 06 00  |...|.........!..|
00000020  00 be be 07 38 04 75 0b  83 c6 10 81 fe fe 07 75  |....8.u........u|
00000030  f3 eb 16 b4 02 b0 01 bb  00 7c b2 80 8a 74 01 8b  |.........|...t..|
00000040  4c 02 cd 13 ea 00 7c 00  00 eb fe 00 00 00 00 00  |L.....|.........|
00000050  00 00 00 00 00 00 00 00  00 00 00 80 01 00 00 00  |................|
00000060  00 00 00 00 ff fa 90 90  f6 c2 80 74 05 f6 c2 70  |...........t...p|
00000070  74 02 b2 80 ea 79 7c 00  00 31 c0 8e d8 8e d0 bc  |t....y|..1......|
00000080  00 20 fb a0 64 7c 3c ff  74 02 88 c2 52 be 05 7c  |. ..d|<.t...R..||
00000090  b4 41 bb aa 55 cd 13 5a  52 72 3d 81 fb 55 aa 75  |.A..U..ZRr=..U.u|
000000a0  37 83 e1 01 74 32 31 c0  89 44 04 40 88 44 ff 89  |7...t21..D.@.D..|
000000b0  44 02 c7 04 10 00 66 8b  1e 5c 7c 66 89 5c 08 66  |D.....f..\|f.\.f|
000000c0  8b 1e 60 7c 66 89 5c 0c  c7 44 06 00 70 b4 42 cd  |..`|f.\..D..p.B.|
000000d0  13 72 05 bb 00 70 eb 76  b4 08 cd 13 73 0d 5a 84  |.r...p.v....s.Z.|
000000e0  d2 0f 83 de 00 be 85 7d  e9 82 00 66 0f b6 c6 88  |.......}...f....|
000000f0  64 ff 40 66 89 44 04 0f  b6 d1 c1 e2 02 88 e8 88  |d.@f.D..........|
00000100  f4 40 89 44 08 0f b6 c2  c0 e8 02 66 89 04 66 a1  |.@.D.......f..f.|
00000110  60 7c 66 09 c0 75 4e 66  a1 5c 7c 66 31 d2 66 f7  |`|f..uNf.\|f1.f.|
00000120  34 88 d1 31 d2 66 f7 74  04 3b 44 08 7d 37 fe c1  |4..1.f.t.;D.}7..|
00000130  88 c5 30 c0 c1 e8 02 08  c1 88 d0 5a 88 c6 bb 00  |..0........Z....|
00000140  70 8e c3 31 db b8 01 02  cd 13 72 1e 8c c3 60 1e  |p..1......r...`.|
00000150  b9 00 01 8e db 31 f6 bf  00 80 8e c6 fc f3 a5 1f  |.....1..........|
00000160  61 ff 26 5a 7c be 80 7d  eb 03 be 8f 7d e8 34 00  |a.&Z|..}....}.4.|
00000170  be 94 7d e8 2e 00 cd 18  eb fe 47 52 55 42 20 00  |..}.......GRUB .|
00000180  47 65 6f 6d 00 48 61 72  64 20 44 69 73 6b 00 52  |Geom.Hard Disk.R|
00000190  65 61 64 00 20 45 72 72  6f 72 0d 0a 00 bb 01 00  |ead. Error......|
000001a0  b4 0e cd 10 ac 3c 00 75  f4 c3 00 00 00 00 00 00  |.....<.u........|
000001b0  00 00 00 00 00 00 00 00  f8 f3 bf 34 00 00 80 04  |...........4....|
000001c0  01 04 83 fe c2 ff 00 08  00 00 00 00 20 00 00 fe  |............ ...|
000001d0  c2 ff 8e fe c2 ff 00 08  20 00 00 f8 df 04 00 00  |........ .......|
000001e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200
```
直接覆盖写入硬盘的前446字节：
`dd if=/dev/zero of=/dev/sda bs=1 count=446`

覆盖写入后，信息如下：
```
hexdump -C -n 512 /dev/sda

00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 80 04  |................|
000001c0  01 04 83 fe c2 ff 00 08  00 00 00 00 20 00 00 fe  |............ ...|
000001d0  c2 ff 8e fe c2 ff 00 08  20 00 00 f8 df 04 00 00  |........ .......|
000001e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200
```

## 引导记录损坏表现
硬盘的引导记录损坏后，当机器在硬件自检完毕，BIOS把控制器转给启动程序，存储设备的启动顺序一般在BIOS的BOOT目录下设置。
但因为硬盘的引导记录已经损坏，所以机器会不断的循环各个存储设备，不断的寻找可引导的启动程序，具体表现如下：
![rehel_post5](/images/post/rhel_post5.png "rehel")

## 修复引导记录
和上面boot目录损坏，grub损坏修复一样，甚至更简单。
同上使用原版镜像进入救援系统，获得shell，修改层次结构到本地系统后
`grub2-install /dev/sda`
重启恢复，结束。

# /etc/fstab文件修复

/etc/fstab是用来存放文件系统的静态信息的文件。位于/etc/目录下，可以用命令cat /etc/fstab来查看，如果要修改的话，则用命令 vi /etc/fstab 来修改。
当系统启动的时候，系统会自动地从这个文件读取信息，并且会自动将此文件中指定的文件系统挂载到指定的目录。

## 误删/etc/fstab
误删fstab文件在重启系统并通过终端登陆（注意这里说的是终端登陆而不是虚拟终端pty，例如ssh登陆！）后会给出只读文件系统的提示：
`-bash: cannot create temp file for here-document: Read-only file system`
也可能没有上面的提示，但是在对文件或目录进行写操作的时候会给出Read-only file system的提示。

## 临时解决方法
单根据上面的提示，我们可以直接通过重新将根目录挂载为可读写来解决，但这只是临时且不负责的做法：
`mount -o remount,rw /`

## 查看/etc/fstab
我这里Redhat8系统下的fstab文件内容如下：
```
cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Thu Aug 15 07:41:14 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
#
# After editing this file, run 'systemctl daemon-reload' to update systemd
# units generated from this file.
#
/dev/mapper/rhel-root   /                       xfs     defaults        0 0
UUID=4c1d4102-5e3a-4b18-8d4d-004ee9eee149 /boot                   xfs     defaults        0 0
/dev/mapper/rhel-swap   swap                    swap    defaults        0 0
```

## 字段解释
/etc/fstab文件主要包括6段，依次是：
```
<file system>　　<dir>　　<type>　　<options>　　<dump>　　<pass>

<file system> 要挂载的分区或存储设备

<dir> 挂载的目录位置

<type> 挂载分区的文件系统类型，比如：ext3、ext4、xfs、swap

<options> 挂载使用的参数有哪些。举例如下：
	auto - 在启动时或键入了 mount -a 命令时自动挂载。
	noauto - 只在你的命令下被挂载。
	exec - 允许执行此分区的二进制文件。
	noexec - 不允许执行此文件系统上的二进制文件。
	ro - 以只读模式挂载文件系统。
	rw - 以读写模式挂载文件系统。
	user - 允许任意用户挂载此文件系统，若无显示定义，隐含启用 noexec, nosuid, nodev 参数。
	users - 允许所有 users 组中的用户挂载文件系统.
	nouser - 只能被 root 挂载。
	owner - 允许设备所有者挂载.
	sync - I/O 同步进行。
	async - I/O 异步进行。
	dev - 解析文件系统上的块特殊设备。
	nodev - 不解析文件系统上的块特殊设备。
	suid - 允许 suid 操作和设定 sgid 位。这一参数通常用于一些特殊任务，使一般用户运行程序时临时提升权限。
	nosuid - 禁止 suid 操作和设定 sgid 位。
	noatime - 不更新文件系统上 inode 访问记录，可以提升性能。
	nodiratime - 不更新文件系统上的目录 inode 访问记录，可以提升性能(参见 atime 参数)。
	relatime - 实时更新 inode access 记录。只有在记录中的访问时间早于当前访问才会被更新。（与 noatime 相似，但不会打断如 mutt 或其它程序探测文件在上次访问后是否被修改的进程。），可以提升性能。
	flush - vfat 的选项，更频繁的刷新数据，复制对话框或进度条在全部数据都写入后才消失。
	defaults - 使用文件系统的默认挂载参数，例如 ext4 的默认参数为:rw, suid, dev, exec, auto, nouser, async.

<dump> dump 工具通过它决定何时作备份. dump 会检查其内容，并用数字来决定是否对这个文件系统进行备份。 允许的数字是 0 和 1 。0 表示忽略， 1 则进行备份。大部分的用户是没有安装 dump 的 ，对他们而言 <dump> 应设为 0。

<pass> fsck 读取 <pass> 的数值来决定需要检查的文件系统的检查顺序。允许的数字是0, 1, 和2。 根目录应当获得最高的优先权 1, 其它所有需要被检查的设备设置为 2. 0 表示设备不会被 fsck 所检查。
```

## 查看分区UUID和文件系统类型
使用blkid命令可以看到所以存储设备的UUID以及其分区的文件系统类型，记住对应的文件系统类型因为这在后面重新编辑fstab文件很重要。
```
blkid

/dev/sda1: UUID="4c1d4102-5e3a-4b18-8d4d-004ee9eee149" TYPE="xfs"
/dev/sda2: UUID="aaoTPj-JNcG-XSRz-QPoE-6Skl-OAiz-l29Y12" TYPE="LVM2_member"
/dev/sr0: UUID="2019-04-04-08-40-23-00" LABEL="RHEL-8-0-0-BaseOS-x86_64" TYPE="iso9660" PTUUID="0da1aba4" PTTYPE="dos"
/dev/mapper/rhel-root: UUID="658a4efe-d92e-4027-a312-7bd0d01aaca7" TYPE="xfs"
/dev/mapper/rhel-swap: UUID="73c88283-5c4d-4a88-b201-3941ba60f43b" TYPE="swap"
```

## 重新编辑fstab
每个人的系统环境不同所以其对应的fstab文件内容也是不一样的，可能很复杂也可能很简单，这里假设用最小化挂载，也就是只挂载/boot、/、swap，其他的分区目录都可以在进入系统后手动挂载。
刚刚使用blkid命令后我们已经可以看到了本地系统下所有的存储设备，将其写入到fstab文件即可，如果你的分区太多以至于不知道哪个是根目录那个是boot目录，那么可以直接使用lsblk或是df命令查看挂载信息。如果你是在救援模式下，那么可以将其各个设备分别挂载到临时目录下，通过查看临时目录中的文件来判断。
```
vi /etc/fstab

/dev/sda1 /boot xfs defaults 0 0
/dev/mapper/rhel-root / xfs defaults 0 0
/dev/mapper/rhel-swap swap swap defaults 0 0
```
我这里根目录和swap分区都是通过LVM创建的，对于LVM不了解的可以查看对应的文档。

# UEFI固件修复引导
关于UEFI引导方式这篇文章中不作介绍，想了解的请百度。

## grubx64.efi文件损坏
如果是开机后提示Faile to open \EFI\redhat\grubx64.efi，如下所示：
```
System BootOrder not fount. Initializing defaults.

Failed to open \EFI\redhat\grubx64.efi - Not Found
Failed to load image \EFI\redhat\grubx64.efi: Not Found
start_image() returned Not found
StartImage failed: Not Found
_
```
可以简单的通过进入原版镜像的救援系统，进入方式请查看上面的GRUB修复。
这里假设你已经进入到了救援系统，并且获得了shell，注意，这里不需要修改层次结构。
```
find / -name "grubx64.efi"

/run/install/repo/EFI/BOOT/grubx64.efi
#在救援系统中查找grubx64.efi所在的路径。

cp /run/install/repo/EFI/BOOT/grubx64.efi /mnt/sysimage/boot/efi/EFI/redhat/
exit
init 6
```
## /boot/efi/EFI/目录丢失
表现为重启后机器无限重启，
假设你已经进入到了救援系统，并且获得了shell，这里需要修改层次结构。
```
chroot /mnt/sysimage
dnf reinstall grub2-efi shim grub2-tools
# 注意我这里是使用的dnf命令，这是rhel8的包管理工具，你也可以使用yum，不过它现在是dnf的一个软链接。
----------------------------------------
# 如果这里你遇到了如下报错
Package shim available, but not installed.
No match for argument: shim
Error: No packages marked for reinstall

# 这里的包名可能是其他的另外两个包，如果你遇到了这个错误，请尝试先卸载包，再安装，而不是reinstall。
dnf remove shim
dnf install shim
----------------------------------------

ls /boot/efi/EFI/redhat
# 可以看到该路径下已经恢复了一些文件，但还不是全部。

find / -name "grubx64.efi"
/run/install/repo/EFI/BOOT/grubx64.efi
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
exit
cp /run/install/repo/EFI/BOOT/grubx64.efi /mnt/sysimage/boot/efi/EFI/redhat/
init 6
```
