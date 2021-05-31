---
title: storcli64常用配置
date: 2019-08-25 11:16:42
tags: [raid,storcli,lsi]
categories: RAID
---

<center>
使用storcli命令在系统下配置RAID<br/>
适应设备不在身边或不能重启的场景
</center>
<!--more-->

# 简单说明
根据官方文档，该命令适用于12Gb/s MegaRAID三模式，具体兼容到什么程度不是很清楚，因为我没有那么多型号的阵列卡来测试。
我这里是使用的家用PC机接的阵列卡，型号是INSPUR 2208MR接了缓存卡和电池，硬盘背板型号是CVPM02，供电线是自己改的，改为了三个大4PIN，也就是我们日常说的大D供电口，一般给机箱风扇或者转给硬盘供电的那种接口。
我测试环境是WINDOWS10，所有使用的storcli也是在DOS环境下的而不是LINUX。

这篇文章里使用的版本为2019/06/07所更新的版本。
[最新版本的下载以及用户手册可以点击这个链接](https://www.broadcom.cn/support/download-search?pg=Storage+Adapters,+Controllers,+and+ICs&pf=RAID+Controller+Cards&pn=&pa=&po=&dk=storcli)

# 查看所有阵列卡及硬盘信息
```
.\storcli64.exe show

CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Status Code = 0
Status = Success
Description = None

Number of Controllers = 1
Host Name = DESKTOP-LCVEENV
Operating System  = Windows 10

System Overview :
===============

-------------------------------------------------------------------------
Ctl Model        Ports PDs DGs DNOpt VDs VNOpt BBU sPR DS  EHS ASOs Hlth
-------------------------------------------------------------------------
  0 INSPUR2208MR     8   5   1     0   1     0 Opt On  1&2 Y      4 Opt
-------------------------------------------------------------------------

Ctl=Controller Index|DGs=Drive groups|VDs=Virtual drives|Fld=Failed
PDs=Physical drives|DNOpt=Array NotOptimal|VNOpt=LD NotOptimal|Opt=Optimal
Msng=Missing|Dgd=Degraded|NdAtn=Need Attention|Unkwn=Unknown
sPR=Scheduled Patrol Read|DS=DimmerSwitch|EHS=Emergency Spare Drive
Y=Yes|N=No|ASOs=Advanced Software Options|BBU=Battery backup unit
Hlth=Health|Safe=Safe-mode boot
```
例如我这里的阵列卡型号为INSPUR2208MR


```
.\storcli64.exe /call show

.\storcli64.exe /call show all
```
这个命令可以显示出阵列卡、缓存卡、BBU、硬盘背板和所有硬盘的信息，加上all参数后显示的信息更为全面，可以对比来看。

# 查看硬盘信息
```
.\storcli64.exe /c0/e252/s5 show
.\storcli64.exe /c0/e252/s5 show all
```
该命令显示具体一块硬盘的详细信息，具体各个配置我也不全了解，可以查看官方的手册。
这里解释下/c0/e252/e5参数的组合
/c0表示第一块阵列卡，如果你的设备上有多张阵列卡，那么后面可能还有/c1、/c2等等。
/e252表示EID，具体含义不了解。
/s5表示slot5这个槽位的硬盘。
查看这个组合同样是使用/call show all这个命令。
```
.\storcli64.exe /call show all

Basics :
======
Controller = 0
Model = INSPUR 2208MR
Serial Number = FW-24GRXAHRFQEWM
Current Controller Date/Time = 08/25/2019, 12:00:15
Current System Date/time = 08/25/2019, 12:00:15
SAS Address = 56c92bf0000b8dc6
PCI Address = 00:01:00:00
Mfg Date = 00/00/00
Rework Date = 00/00/00
Revision No =

这里的Controller的值为0，表示第一张阵列卡，所以使用/c0

PD LIST :
=======

--------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model              Sp Type
--------------------------------------------------------------------------------
252:3    20 JBOD  -    3.638 TB SATA HDD N   N  512B ST4000DM000-1F2168 U  -
252:4    19 JBOD  -  558.911 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:5    18 JBOD  -  558.911 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:6    17 JBOD  -  558.911 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:7    16 JBOD  -  558.911 GB SAS  HDD N   N  512B AL13SEB600         U  -
--------------------------------------------------------------------------------

这里EID:Slt分别为/e252/sx，x值视你需要操作的硬盘而定，比如我想查看slot6这块硬盘，那么使用如下命令：
.\storcli64.exe /c0/e252/s6 show all
```


主要查看首段的四行配置
```
Shield Counter = 0
Media Error Count = 0
Other Error Count = 0
BBM Error Count = 0
```
如果这四行配置值都为0，那么这块硬盘基本是正常可用的，如果这四行配置的值不为0而为其他值的话，那么请小心了，这块硬盘可能已经损害或者处在损坏的边缘随时会被踢出RAID了。

加all参数后显示的信息更为全面，建议加all查看。

# T-10 PI（数据保护）
T-10 PI（数据保护）下面简称PI
[具体官方介绍可以点击查看这个链接](https://www.broadcom.com/support/knowledgebase/1211161502244/the-data-protection-feature---megaraid)
这里给PI功能单开了一个标题，因为这里需要特别注意：PI功能是需要阵列卡和硬盘驱动器在硬件上双向支持且在软件上开启PI功能，否则创建RAID组选择开启PI是无法成功创建的。
对于阵列卡和磁盘驱动器都开启PI功能后创建的RAID组具体功能如何我不太了解，但后期此RAID组更换硬盘也是需要同样开启PI的硬盘，不论RAID的等级，都需要PI功能的支持，比如开启PI功能的RAID5当损坏了一块硬盘后更换上一块未开启PI的硬盘后，服务器和硬盘的告警都会消失但该盘无法加入到RAID组中！

## 开启阵列卡的PI功能
使用下面的命令可以开启阵列卡的PI功能：
```
.\storcli64.exe /c0 set pi state=on

CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

----------------
Ctrl_Prop Value
----------------
Enable PI ON
----------------
#一般阵列卡是默认开启PI功能的。

#查看阵列卡的PI信息
.\storcli64.exe /c0 show pi

CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

-----------------
Ctrl_Prop  Value
-----------------
PI Support ON
PI Import  OFF
-----------------
```

# JBOD
JBOD（Just a bunch of disk）严格上来说不是一种RAID，因为它只是简单将多个磁盘合并成一个大的逻辑盘，并没有任何的数据冗余。数据的存放机制就是从第一块磁盘开始依序向后存储数据。如果某个磁盘损毁，则该盘上的数据就会丢失，但不会丢失所有数据。
在我看来

## 设置阵列卡为JBOD模式

设置阵列卡为JBOD模式，执行该命令后会将该阵列卡下所有UG盘设置为JBOD状态，而原本的RAID组是不受影响的。
在JBOD模式下一样可以将硬盘改为UG状态后创建RAID组或将其加入到一个RAID中。

```
#设置阵列卡为JBOD模式，执行该命令后会将该阵列卡下所有UG盘设置为JBOD状态。
.\storcli64.exe /c0 set jbod=on

#可以通过下面的命令查看硬盘状态
.\storcli64.exe /c0 show all

```
现在你可以在系统下看到一些未初始化的硬盘了，windows下可以通过磁盘管理查看、linux可以通过lsblk或fdisk查看。
现在这些硬盘是无法使用的，windows下无法分配盘符，linux下表现为无法挂载，原因是硬盘还没有指定文件系统。
windows下在磁盘管理中可以初始化，linux下可以指定一个文件系统，如mkfs.ext4命令。

## 修改硬盘为UG状态
将硬盘设置为UG状态后可以对该盘进行RAID配置，使用set命令来修改硬盘状态,假设这里我将/c0/e252/s6这块盘由JBOD改为UG状态：
```
.\storcli64.exe /c0/e252/s6 set good
#如果执行该命令失败，提示Failure，则添加force参数强制执行。
.\storcli64.exe /c0/e252/s5 set good force

.\storcli64.exe /c0/e252/s6 set ?
#可以查看所有可以设置的状态。
```
# 配置RAID
storcli命令支持raid[0|1|5|6|00|10|50|60]，以下例子将添加一个RAID5卷
```
.\storcli64.exe /c0 add vd raid5 size=300GB drives=252:4-6 spares=252:7
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Add VD Succeeded.

#查看刚刚创建的VD0，默认创建后的VD值是按升序顺次命名的。
.\storcli64.exe /c0/v0 show
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Virtual Drives :
==============

-------------------------------------------------------------
DG/VD TYPE  State Access Consist Cache Cac sCC     Size Name
-------------------------------------------------------------
0/0   RAID5 Optl  RW     No      RWBD  -   ON  300.000 GB
-------------------------------------------------------------

EID=Enclosure Device ID| VD=Virtual Drive| DG=Drive Group|Rec=Recovery
Cac=CacheCade|OfLn=OffLine|Pdgd=Partially Degraded|Dgrd=Degraded
Optl=Optimal|RO=Read Only|RW=Read Write|HD=Hidden|TRANS=TransportReady|B=Blocked|
Consist=Consistent|R=Read Ahead Always|NR=No Read Ahead|WB=WriteBack|
AWB=Always WriteBack|WT=WriteThrough|C=Cached IO|D=Direct IO|sCC=Scheduled
Check Consistency
```
这里例子中创建了一个有四块硬盘RAID5，其中一块硬盘为热备盘且指定了容量为300GB，如果提示创建失败，请检查指定的磁盘驱动器状态是否都为UG，否则请检查参数是否符合规范，可以通过如下命令展示参数格式。
`.\storcli64.exe add vd ?`



|参数|取值范围|描述|
|:-:|:-:|:-:|
|raid|[0、1、5、6、00、10、50、60]|设置配置的RAID类型。|
|size|MAX值取决于RAID组中的硬盘容量和RAID级别|设置RAID组（VD）的最大容量，默认值是所有引用硬盘的最大容量|
|name|长度为15个字符|指定RAID组的卷组名称|
|drives|需要创建RAID组的硬盘槽位|e:s、e:s-x、e:s-x,y:<br/>  - e指定硬盘柜ID<br/>  - s表示在盘柜中的槽位|
|pdperarray|1-16|指定每个阵列的物理驱动器数，该默认值自动选择。|
|SED|-|创建启用安全性的驱动器。|
|pdcache|ON、OFF、默认|启用或禁用PD缓存。|
|PI|-|启用保护信息。|
|dimmerswitch|default：逻辑设备使用控制器默认节电政策。<br/>automatic（自动）：逻辑设备电源节省由固件管理。<br/>none：没有节电政策。<br/>maximum(max)：逻辑设备使用最大值省电。<br/>MaximumWithoutCaching（maxnocache）：逻辑设备不缓存写入，最大化节能。|指定节电策略。自动设置为默认值（default）。|
|direct、cached|direct：缓存的I/O。<br/>cached：直接I/O|设置逻辑驱动器高速缓存策略。直接I/O是默认值。|
|EmulationType|0：默认仿真，这意味着有配置的ID中的任何512e驱动器，然后是每个扇区的物理字节显示为512e（4k）。如果没有512e驱动器的物理字节每个扇区将是512n。<br/>1：禁用，这意味着即使有配置的ID中没有512e驱动器每个扇区的物理字节将显示为512n。<br/>2：强制，这意味着即使有配置的ID中没有512e驱动器每个扇区的物理字节将显示为512e（4K）。|默认值为0|
|wt、wb、awb|wt：Write through。<br/>wb：回写。<br/>awb：总是回写。|允许直写，默认值是wb|
|nora、ra|ra：预读。<br/>nora：不要预读。|禁用预读，默认值为ra|
|cachevd|-|在创建的虚拟驱动器上启用SSD缓存。|
|strip|8,16,32,64,128,256,512,1024<br/>注意：支持的条带大小可能不同MegaRAID至少为64 KB到1 MB控制器和集成的64 KBMegaRAID控制器。|设置RAID配置的条带大小。|
|aftervd|有效的VD值（RAID组号）。|在指定VD旁边的相邻空闲插槽中创建VD。|
|spares|有效的局部热备盘数量|指定需要分配给此RAID组的热备盘数量。|
|force|-|强制将具有安全功能的物理驱动器添加到未开启安全功能的RAID组中。|

## 配置RAID失败(2021.05.31记)

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0 add vd r1 drives=0:6,10
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Failure
Description = resources already in use
```

我们查看下`s6`和`s10`这两块盘的状态

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0 show

PD LIST :
=======

------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model            Sp Type
------------------------------------------------------------------------------
0:0      14 Onln   0 558.406 GB SAS  HDD N   N  512B HUC101860CSS200  U  -
0:4      13 Onln   0 558.406 GB SAS  HDD N   N  512B AL14SEB060N      U  -
0:6      10 UGood  F   3.637 TB SAS  HDD N   N  512B ST4000NM0023     U  -
0:10     11 UGood  -   3.637 TB SAS  HDD N   N  512B ST4000NM0023     D  -
------------------------------------------------------------------------------
```

可以看到，这里的`s10`的`Spun`状态为`DOWN`，我们尝试将其设置为`UP`

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 spinup
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Failure
Description = Spin Up Drive Failed.

Detailed Status :
===============

------------------------------------------------------------------------
Drive      Status  ErrCd ErrMsg
------------------------------------------------------------------------
/c0/e0/s10 Failure    50 device state doesn't support requested command
------------------------------------------------------------------------
```

提示错误，继续尝试清理`PD`硬盘状态：

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 start erase simple
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Start Drive Erase Succeeded

root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 show erase
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Show Drive Erase Status Succeeded.


-----------------------------------------------------
Drive-ID   Progress% Status      Estimated Time Left
-----------------------------------------------------
/c0/e0/s10         3 In progress -
-----------------------------------------------------

# earse 除了擦除 PD 状态，还会擦除数据，4T 硬盘耗时还是比较久的，且擦盘期间IO异常的慢，建议空闲时进行，可以先stop
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 stop erase
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Stop Drive Erase Succeeded.

# 因为我只需要他的Spin状态复位就行了， 查看盘的Sp状态为U，就可以 stop erase 了，不用等擦除全部跑完
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 show
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Show Drive Information Succeeded.


Drive Information :
=================

----------------------------------------------------------------------------
EID:Slt DID State DG     Size Intf Med SED PI SeSz Model            Sp Type
----------------------------------------------------------------------------
0:10     11 UGood -  3.637 TB SAS  HDD N   N  512B ST4000NM0023     U  -
----------------------------------------------------------------------------

# 注意，sp状态为UP后，将其down，再up一次，否则可能过会就自己down了.
# down
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 spindown
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Spin Down Drive Succeeded.

# up
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/e0/s10 spinup
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Spin Up Drive Succeeded.
```

解决了`s10`盘`sp`状态`down`问题后，我们继续解决`s6`的`DG`字段为`F`（存在外部`RAID`配置）的问题：

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0 show

PD LIST :
=======

------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model            Sp Type
------------------------------------------------------------------------------
0:0      14 Onln   0 558.406 GB SAS  HDD N   N  512B HUC101860CSS200  U  -
0:4      13 Onln   0 558.406 GB SAS  HDD N   N  512B AL14SEB060N      U  -
0:6      10 UGood  F   3.637 TB SAS  HDD N   N  512B ST4000NM0023     U  -
0:10     11 UGood  -   3.637 TB SAS  HDD N   N  512B ST4000NM0023     U  -
------------------------------------------------------------------------------
```

可以看到`s6`的DG为F， 表示有外部的`RAID`配置信息，应该是之前的`RAID`信息没有清理，我们使用`fall`查看下：

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/fall show
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Operation on foreign configuration Succeeded


FOREIGN CONFIGURATION :
=====================

----------------------------------------
DG EID:Slot Type   State     Size NoVDs
----------------------------------------
 0 -        Nytro1 Frgn  3.637 TB     1
----------------------------------------

NoVDs - Number of VD in Drive Group
DG=Disk Group Index|Arr=Array Index|Row=Row Index|EID=Enclosure Device ID
DID=Device ID|Type=Drive Type|Onln=Online|Rbld=Rebuild|Optl=Optimal|Dgrd=Degraded
Pdgd=Partially degraded|Offln=Offline|BT=Background Task Active
PDC=PD Cache|PI=Protection Info|SED=Self Encrypting Drive|Frgn=Foreign
DS3=Dimmer Switch 3|dflt=Default|Msng=Missing|FSpace=Free Space Present
TR=Transport Ready

Total foreign Drive Groups = 1
```

回显中有一个外部的`RAID`信息，使用命令`storcli /c0/fall delete`删除掉控制器c0的所有外部配置信息，删除后DG状态为空，可以正常使用：

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0/fall delete
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Successfully deleted foreign configuration

再次`show`一下：

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0 show

PD LIST :
=======

------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model            Sp Type
------------------------------------------------------------------------------
0:0      14 Onln   0 558.406 GB SAS  HDD N   N  512B HUC101860CSS200  U  -
0:4      13 Onln   0 558.406 GB SAS  HDD N   N  512B AL14SEB060N      U  -
0:6      10 UGood  -   3.637 TB SAS  HDD N   N  512B ST4000NM0023     U  -
0:10     11 UGood  -   3.637 TB SAS  HDD N   N  512B ST4000NM0023     U  -
------------------------------------------------------------------------------

OK，两块盘都没问题了， 尝试做`raid1`：

```shell
root@ubuntu:/opt/MegaRAID/storcli# ./storcli64 /c0 add vd r1 drives=0:6,10
CLI Version = 007.1705.0000.0000 Mar 31, 2021
Operating system = Linux 4.4.0-131-generic
Controller = 0
Status = Success
Description = Add VD Succeeded.

#出现了sdc，分区信息不用管
root@ubuntu:/opt/MegaRAID/storcli# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   7.3T  0 disk
|-sda1   8:1    0   128M  0 part
`-sda2   8:2    0   7.3T  0 part /data
sdb      8:16   0 558.4G  0 disk
|-sdb1   8:17   0   953M  0 part /boot
`-sdb2   8:18   0 557.5G  0 part /
sdc      8:32   0   3.7T  0 disk
|-sdc1   8:33   0   100M  0 part
`-sdc2   8:34   0 278.8G  0 part

root@ubuntu:/opt/MegaRAID/storcli# fdisk -l /dev/sdc
Disk /dev/sdc: 3.7 TiB, 4000225165312 bytes, 7812939776 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

配置文件系统，挂载结束（如果担心，可以先用fdisk或者看下）:

```shell
root@ubuntu:/opt/MegaRAID/storcli# fdisk -l /dev/sdc
Disk /dev/sdc: 3.7 TiB, 4000225165312 bytes, 7812939776 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
# 无分区信息

root@ubuntu:/opt/MegaRAID/storcli# parted -l
Model: JMicron Tech (scsi)
Disk /dev/sda: 8002GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                          Flags
 1      17.4kB  134MB   134MB                Microsoft reserved partition  msftres
 2      135MB   8002GB  8001GB  ntfs         Basic data partition          msftdata


Model: LSI MR ROMB (scsi)
Disk /dev/sdb: 600GB
Sector size (logical/physical): 512B/4096B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size   Type     File system  Flags
 1      1049kB  1000MB  999MB  primary  ext4
 2      1000MB  600GB   599GB  primary  ext4         boot


Error: /dev/sdc: unrecognised disk label
Model: LSI MR ROMB (scsi)
Disk /dev/sdc: 4000GB
Sector size (logical/physical): 512B/4096B
Partition Table: unknown
Disk Flags:
# 无分区信息

root@ubuntu:/opt/MegaRAID/storcli# blkid /dev/sdc
# 同样无输出，没有分区信息，放心的做文件系统吧

root@ubuntu:/opt/MegaRAID/storcli# mkfs.ext4 /dev/sdc
mke2fs 1.42.13 (17-May-2015)
Creating filesystem with 976617472 4k blocks and 244154368 inodes
Filesystem UUID: 86a425b2-8853-49af-8b03-bf9844736ea9
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

root@ubuntu:/opt/MegaRAID/storcli# mkdir /data2
root@ubuntu:/opt/MegaRAID/storcli# mount /dev/sdc /data2
root@ubuntu:/opt/MegaRAID/storcli# ls /data2
lost+found
root@ubuntu:/opt/MegaRAID/storcli# df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs   63G     0   63G   0% /dev
tmpfs          tmpfs      13G   74M   13G   1% /run
/dev/sdb2      ext4      549G  2.8G  518G   1% /
tmpfs          tmpfs      63G     0   63G   0% /dev/shm
tmpfs          tmpfs     5.0M     0  5.0M   0% /run/lock
tmpfs          tmpfs      63G     0   63G   0% /sys/fs/cgroup
/dev/sdb1      ext4      922M   59M  801M   7% /boot
tmpfs          tmpfs      13G     0   13G   0% /run/user/0
/dev/sda2      fuseblk   7.3T  3.3T  4.1T  45% /data
/dev/sdc       ext4      3.6T   68M  3.4T   1% /data2
```

**2021.05.31 完工！**



# 扩展RAID组容量

这里刚刚创建了一个容量为300GB的RAID5没有使用全部的硬盘容量，所以可以进行扩展：
以下的例子我将其扩大800GB，好像没有找到参数可以直接扩展为RAID的所有容量：
```
.\storcli64.exe /c0/v0 expand size=800GB
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = expansion operation succeeded


EXPANSION RESULT :
================

----------------------------------------------------------------------------------
VD       Size FreSpc     ReqSize    AbsUsrSz   %FreSpc NewSize  Status NoArrExp
----------------------------------------------------------------------------------
 0 300.000 GB 816.812 GB 800.000 GB 800.475 GB      98 1.074 TB -      816.812 GB
----------------------------------------------------------------------------------

Size - Current VD size|FreSpc - Freespace available before expansion
%FreSpc - Requested expansion size in % of available free space
AbsUsrSz - User size rounded to nearest %
```
storcli /cx/vx show expansion
此命令显示有和没有阵列扩展的虚拟驱动器上的扩展信息
```
.\storcli64.exe /c0/v0 show expansion
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


EXPANSION INFORMATION :
=====================

-------------------------------------------
VD     Size OCE NoArrExp WithArrExp Status
-------------------------------------------
 0 1.074 TB N   -        -          -
-------------------------------------------

OCE - Online Capacity Expansion | WithArrExp - With Array Expansion
 VD=Virtual Drive | NoArrExp - Without Array Expansion
```
# 在原RAID组增加新盘（RAID迁移）
在已存在的RAID中添加新硬盘，扩大RAID容量，当增加新盘后，可进行RAID级别迁移。
storcli /cx/vx start migrate <type=raidx> [option=<add|remove> drives=[e:]s|[e:]s-x|[e:]s-x,y] [Force]
首先创建一个单盘RAID0，接下来对其进行添盘操作，在此RAID0中添加一个slot5新盘：

```
.\storcli64.exe /c0/v0 start migrate type=raid1 option=add drives=252:5
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Start Reconstruction Operation Success

#type 增加硬盘后RAID的级别
- 与原RAID级别相同时，该命令为扩大RAID容量。
- 与原RAID级别不同时，该命令为RAID级别迁移。
```
刚刚在VD0中添加了一块新盘，并将其迁移至RAID1。



# 查看所有RAID组
这里查看所有RAID组需要指定阵列卡，例如我这里只有一张阵列卡，那么就是/c0/vall show
```
.\storcli64.exe /c0/vall show all
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


/c0/v0 :
======

-------------------------------------------------------------
DG/VD TYPE  State Access Consist Cache Cac sCC     Size Name
-------------------------------------------------------------
0/0   RAID5 Optl  RW     No      RWBD  -   ON  1.089 TB
-------------------------------------------------------------

EID=Enclosure Device ID| VD=Virtual Drive| DG=Drive Group|Rec=Recovery
Cac=CacheCade|OfLn=OffLine|Pdgd=Partially Degraded|Dgrd=Degraded
Optl=Optimal|RO=Read Only|RW=Read Write|HD=Hidden|TRANS=TransportReady|B=Blocked|
Consist=Consistent|R=Read Ahead Always|NR=No Read Ahead|WB=WriteBack|
AWB=Always WriteBack|WT=WriteThrough|C=Cached IO|D=Direct IO|sCC=Scheduled
Check Consistency


PDs for VD 0 :
============

------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model            Sp Type
------------------------------------------------------------------------------
252:4    19 Onln   0 558.406 GB SAS  HDD N   N  512B AL13SEB600       U  -
252:5    18 Onln   0 558.406 GB SAS  HDD N   N  512B AL13SEB600       U  -
252:6    17 Onln   0 558.406 GB SAS  HDD N   N  512B AL13SEB600       U  -
------------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Slot No.|DID=Device ID|DG=DriveGroup
DHS=Dedicated Hot Spare|UGood=Unconfigured Good|GHS=Global Hotspare
UBad=Unconfigured Bad|Onln=Online|Offln=Offline|Intf=Interface
Med=Media Type|SED=Self Encryptive Drive|PI=Protection Info
SeSz=Sector Size|Sp=Spun|U=Up|D=Down|T=Transition|F=Foreign
UGUnsp=Unsupported|UGShld=UnConfigured shielded|HSPShld=Hotspare shielded
CFShld=Configured shielded|Cpybck=CopyBack|CBShld=Copyback Shielded
UBUnsp=UBad Unsupported


VD0 Properties :
==============
Strip Size = 256 KB
Number of Blocks = 2342125568
VD has Emulated PD = No
Span Depth = 1
Number of Drives Per Span = 3
Write Cache(initial setting) = WriteBack
Disk Cache Policy = Disk's Default
Encryption = None
Data Protection = Disabled
Active Operations = None
Exposed to OS = Yes
OS Drive Name = N/A
Creation Date = 30-08-2019
Creation Time = 07:41:30 PM
Emulation type = default
Is LD Ready for OS Requests = Yes
SCSI NAA Id = 66c92bf0000b8dc624fc386ab3141ed3
Unmap Enabled = N/A
```
# 设置RAID的读写策略
```
storcli /cx/vx set accesspolicy=<rw|ro|blocked|rmvblkd>

#例如我这里把刚刚创建的VD0设置为只读模式
.\storcli64.exe /c0/v0 set accesspolicy=ro
```
这里我把RAID组设置为了只读状态，现在在系统下对这个逻辑盘已经无法写入了，但是还可以正常读取。
```
accesspolicy=<rw|ro|blocked|rmvblkd>
- rw 读写
- ro 只读
- blocked 禁止访问
- rmvblkd 删除或阻止
```

# 设置RAID启动项
不知道是否有用，先记录下吧，此命令将物理驱动器设置或取消设置为引导驱动器。
例如，设置VD0为启动项：
`.\storcli64.exe /c0/v0 set bootdrive=on`
设置slot6为启动项：
`.\storcli64.exe /c0/e252/s6 set bootdrive=on`

# 热备硬盘
不用解释作用了，直接上命令，下面的命令用来添加和删除热备盘：
```
.\storcli64.exe /c0/e252/s7 add hotsparedrive
#此命令将/c0/e252/s7设置为全局热备盘

.\storcli64.exe /c0/e252/s7 add hotsparedrive dgs=0
#此命令将/c0/e252/s7设置为VD0的专用热备

.\storcli64.exe /c0/e252/s7 delete hotsparedrive
#此命令将删除/c0/e252/s7的热备属性，无论其是全局热备还是专用热备
```
storcli /cx[/ex]/sx add hotsparedrive {dgs=<n|0,1,2...>}[enclaffinity][nonrevertible]
nonrevertible参数用于指定将硬盘设置为不可逆热备盘，这里文档中对于不可逆热备没有多做说明，猜测可能是指当一个RAID组中有一块硬盘出现故障而被踢出RAID后，热备盘会加入此RAID组进行同步，且原故障盘被替换成一块新盘后，由新盘接替热备属性。
官方文档中对于enclaffinity没看懂。。。具体请参考文档吧。

# 设置紧急热备
设置是否启用紧急热备功能，并设置SMART Error时是否启动紧急热备功能。
storcli /cx set eghs [state=<on|off>][smarter=<on|off>][eug=<on|off>]
```
.\storcli64.exe /c0 show eghs

#显示/c0这块阵列卡的紧急热备状态
```
开启紧急热备功能，并允许硬盘在SMART Error时使用紧急热备功能。
```
.\storcli64.exe /c0 set eghs eug=on
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

------------------
Ctrl_Prop   Value
------------------
EmergencyUG ON
------------------


.\storcli64.exe /c0 set eghs smarter=on
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

-----------------------
Ctrl_Prop        Value
-----------------------
EmergencySmarter ON
-----------------------
```

# 删除RAID
删除RAID信息较为简单，使用/c0/vx del，我这里就删除刚刚创建的VD0。
```
.\storcli64.exe /c0/v0 del
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Delete VD succeeded

#如果这里提示失败，很可能是RAID组中有还在同步的盘，可以加上force参数强制执行，命令如下：
.\storcli64.exe /c0/v0 del force

#删除该阵列卡上所有的RAID组：
.\storcli64.exe /c0/vall del
同样，该命令也可以加上force参数强制执行。

#查看磁盘驱动器状态，已经由Onil改为了UG状态
.\storcli64.exe /c0 show
Generating detailed summary of the adapter, it may take a while to complete.

CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None

Product Name = INSPUR 2208MR
Serial Number = FW-24GRXAHRFQEWM
SAS Address =  56c92bf0000b8dc6
PCI Address = 00:01:00:00
System Time = 08/30/2019 20:08:38
Mfg. Date = 00/00/00
Controller Time = 08/30/2019 20:08:38
FW Package Build = 23.34.0-0014
BIOS Version = 5.50.03.0_4.17.08.00_0x06110200
FW Version = 3.460.65-6268
Driver Name = megasas2i.sys
Driver Version = 6.714.05.00
Vendor Id = 0x1000
Device Id = 0x5B
SubVendor Id = 0x1BD4
SubDevice Id = 0x7
Host Interface = PCI-E
Device Interface = SAS-6G
Bus Number = 1
Device Number = 0
Function Number = 0
JBOD Drives = 1

JBOD LIST :
=========

------------------------------------------------------------------------------
EID:Slt DID State DG     Size Intf Med SED PI SeSz Model              Sp Type
------------------------------------------------------------------------------
252:3    20 JBOD  -  3.638 TB SATA HDD N   N  512B ST4000DM000-1F2168 U  -
------------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Slot No.|DID=Device ID|Onln=Online|
Offln=Offline|Intf=Interface|Med=Media Type|SeSz=Sector Size

Physical Drives = 5

PD LIST :
=======

--------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model              Sp Type
--------------------------------------------------------------------------------
252:3    20 JBOD  -    3.638 TB SATA HDD N   N  512B ST4000DM000-1F2168 U  -
252:4    19 UGood -  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:5    18 UGood -  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:6    17 UGood -  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:7    16 UGood -  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
--------------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Slot No.|DID=Device ID|DG=DriveGroup
DHS=Dedicated Hot Spare|UGood=Unconfigured Good|GHS=Global Hotspare
UBad=Unconfigured Bad|Onln=Online|Offln=Offline|Intf=Interface
Med=Media Type|SED=Self Encryptive Drive|PI=Protection Info
SeSz=Sector Size|Sp=Spun|U=Up|D=Down|T=Transition|F=Foreign
UGUnsp=Unsupported|UGShld=UnConfigured shielded|HSPShld=Hotspare shielded
CFShld=Configured shielded|Cpybck=CopyBack|CBShld=Copyback Shielded
UBUnsp=UBad Unsupported


Cachevault_Info :
===============

------------------------------------
Model  State   Temp Mode MfgDate
------------------------------------
CVPM02 Optimal 33C  -    2014/12/21
------------------------------------
```

# 硬盘数据擦除
当我们使用上面的删除RAID命令后，只是在阵列卡上删除了RAID信息，而硬盘中的RAID信息其实没有丢失的。
虽然我们使用show命令查看硬盘是处于UG状态，它不影响我们对其再配置RAID，但是当我们将它改为JBOD状态的话，其硬盘里存储的RAID信息导致我们无法在系统下进行操作，所有需要清除掉这些数据。
**注意：该操作将会擦除硬盘里的所有数据，如果只是阵列卡RAID信息丢失而硬盘中的RAID信息还在的话，是有机会还原的，执行该擦除命令后就无法恢复了！**

以下例子我将尝试通过storcli /cx[/ex]/sx start erase命令擦除s6这块硬盘
```
#注意，在RAID组中的硬盘是不允许执行擦除操作的，会报错。
.\storcli64.exe /c0/e252/s6 start erase
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Start Drive Erase Succeeded.


#查看擦除进度（剩余时间）
.\storcli64.exe /c0/e252/s6 show erase
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Show Drive Erase Status Succeeded.


------------------------------------------------------
Drive-ID    Progress% Status      Estimated Time Left
------------------------------------------------------
/c0/e252/s6         8 In progress 44 Minutes
------------------------------------------------------


CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Stop Drive Erase Succeeded.
```
storcli /cx[/ex]/sx start erase [simple|normal|crypto|thorough][patternA=<value1>] [patternB=<value2>]
输入示例：
`.\storcli64.exe /c0/e252/s6 start erase thorough patternA=10010011 patternB=11110000`



|参数|取值范围|描述|
|:-:|:-:|:-:|
|erase|simple（简单）: 单通，单模式，对硬盘进行一轮擦除动作<br/>normal（正常）: 三通，三模式，对硬盘进行三轮擦除动作<br/>thorough（彻底）: 九通道，重复正常写入3次，对硬盘进行九轮擦除动作<br/>crypto: 对SSD驱动器执行加密擦除|安全擦除类型|
|patternA|8位值|擦除模式A以覆盖数据|
|patternB|8位值|擦除模式B以覆盖数据|

## 安全擦除
安全擦除仅适用于SED驱动器。
storcli /cx[/ex]/sx secureerase [force]
尝试安全擦拭/c0/e252/s6
```
.\storcli64.exe /c0/e252/s6 secureerase force
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Failure
Description = Drive Secure Erase Failed.

Detailed Status :
===============

---------------------------------------------------------------------
Drive       Status  ErrCd ErrMsg
---------------------------------------------------------------------
/c0/e252/s6 Failure   255 Secure erase is not allowed on this drive.
---------------------------------------------------------------------
```
这里会报错，因为我这里没有SED，不论是否加上了force参数。

# 查看重建进度（设置重建速率）
## 设置重建速率
storcli /cx set rebuildrate=<value>
一般阵列卡默认的重建速率为30%，这里的30%是指占用I/O速率的百分比，高重建速率会降低I/O事务处理速度。
```
.\storcli64.exe /c0 set rebuildrate=50
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

------------------
Ctrl_Prop   Value
------------------
Rebuildrate 50%
------------------

#以百分比显示指定控制器的当前重建任务速率，这里我查看的是第一个阵列卡控制器即/c0
.\storcli64.exe /c0 show rebuildrate
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

------------------
Ctrl_Prop   Value
------------------
Rebuildrate 50%
------------------
```
## 查看硬盘重建进度
查看当前所有硬盘属性
```
.\storcli64.exe /c0 show

Generating detailed summary of the adapter, it may take a while to complete.

CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None

Product Name = INSPUR 2208MR
Serial Number =
SAS Address =  56c92bf0000b8dc6
PCI Address = 00:01:00:00
System Time = 08/31/2019 10:36:03
Mfg. Date = 00/00/00
Controller Time = 08/31/2019 10:36:02
FW Package Build = 23.34.0-0014
BIOS Version = 5.50.03.0_4.17.08.00_0x06110200
FW Version = 3.460.65-6268
Driver Name = megasas2i.sys
Driver Version = 6.714.05.00
Vendor Id = 0x1000
Device Id = 0x5B
SubVendor Id = 0x1BD4
SubDevice Id = 0x7
Host Interface = PCI-E
Device Interface = SAS-6G
Bus Number = 1
Device Number = 0
Function Number = 0
Drive Groups = 1

TOPOLOGY :
========

-----------------------------------------------------------------------------
DG Arr Row EID:Slot DID Type  State BT       Size PDC  PI SED DS3  FSpace TR
-----------------------------------------------------------------------------
 0 -   -   -        -   RAID5 Optl  N    1.089 TB dflt N  N   dflt N      N
 0 0   -   -        -   RAID5 Optl  N    1.089 TB dflt N  N   dflt N      N
 0 0   0   252:4    19  DRIVE Onln  N  558.406 GB dflt N  N   dflt -      N
 0 0   1   252:5    18  DRIVE Onln  N  558.406 GB dflt N  N   dflt -      N
 0 0   2   252:6    17  DRIVE Onln  N  558.406 GB dflt N  N   dflt -      N
-----------------------------------------------------------------------------

DG=Disk Group Index|Arr=Array Index|Row=Row Index|EID=Enclosure Device ID
DID=Device ID|Type=Drive Type|Onln=Online|Rbld=Rebuild|Dgrd=Degraded
Pdgd=Partially degraded|Offln=Offline|BT=Background Task Active
PDC=PD Cache|PI=Protection Info|SED=Self Encrypting Drive|Frgn=Foreign
DS3=Dimmer Switch 3|dflt=Default|Msng=Missing|FSpace=Free Space Present
TR=Transport Ready

Virtual Drives = 1

VD LIST :
=======

-------------------------------------------------------------
DG/VD TYPE  State Access Consist Cache Cac sCC     Size Name
-------------------------------------------------------------
0/0   RAID5 Optl  RW     No      RWBD  -   ON  1.089 TB
-------------------------------------------------------------

EID=Enclosure Device ID| VD=Virtual Drive| DG=Drive Group|Rec=Recovery
Cac=CacheCade|OfLn=OffLine|Pdgd=Partially Degraded|Dgrd=Degraded
Optl=Optimal|RO=Read Only|RW=Read Write|HD=Hidden|TRANS=TransportReady|B=Blocked|
Consist=Consistent|R=Read Ahead Always|NR=No Read Ahead|WB=WriteBack|
AWB=Always WriteBack|WT=WriteThrough|C=Cached IO|D=Direct IO|sCC=Scheduled
Check Consistency

JBOD Drives = 1

JBOD LIST :
=========

------------------------------------------------------------------------------
EID:Slt DID State DG     Size Intf Med SED PI SeSz Model              Sp Type
------------------------------------------------------------------------------
252:3    20 JBOD  -  3.638 TB SATA HDD N   N  512B ST4000DM000-1F2168 U  -
------------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Slot No.|DID=Device ID|Onln=Online|
Offln=Offline|Intf=Interface|Med=Media Type|SeSz=Sector Size

Physical Drives = 5

PD LIST :
=======

--------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model              Sp Type
--------------------------------------------------------------------------------
252:3    20 JBOD  -    3.638 TB SATA HDD N   N  512B ST4000DM000-1F2168 U  -
252:4    19 Onln  0  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:5    18 Onln  0  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:6    17 Onln  0  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:7    16 GHS   -  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
--------------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Slot No.|DID=Device ID|DG=DriveGroup
DHS=Dedicated Hot Spare|UGood=Unconfigured Good|GHS=Global Hotspare
UBad=Unconfigured Bad|Onln=Online|Offln=Offline|Intf=Interface
Med=Media Type|SED=Self Encryptive Drive|PI=Protection Info
SeSz=Sector Size|Sp=Spun|U=Up|D=Down|T=Transition|F=Foreign
UGUnsp=Unsupported|UGShld=UnConfigured shielded|HSPShld=Hotspare shielded
CFShld=Configured shielded|Cpybck=CopyBack|CBShld=Copyback Shielded
UBUnsp=UBad Unsupported


Cachevault_Info :
===============

------------------------------------
Model  State   Temp Mode MfgDate
------------------------------------
CVPM02 Optimal 34C  -    2014/12/21
------------------------------------
```
我这里是使用slot4-6槽位的硬盘做了一组RAID5，slot7为全局热备盘，接下来将slot6盘设置为offline，使热备盘加入RAID组中并自动重建。
```
.\storcli64.exe /c0/e252/s6 set offline
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Set Drive Offline Succeeded.

.\storcli64.exe /c0 show
...
--------------------------------------------------------------------------------
EID:Slt DID State DG       Size Intf Med SED PI SeSz Model              Sp Type
--------------------------------------------------------------------------------
252:3    20 JBOD  -    3.638 TB SATA HDD N   N  512B ST4000DM000-1F2168 U  -
252:4    19 Onln  0  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:5    18 Onln  0  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:6    17 UGood -  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
252:7    16 Rbld  0  558.406 GB SAS  HDD N   N  512B AL13SEB600         U  -
--------------------------------------------------------------------------------
...
```
可以看到现在slot6槽的硬盘已经被我们手动踢出RAID组了，而slot7槽的全局热备盘自动顶上去并进入重建状态了，下面使用storcli /cx[/ex]/sx show rebuild命令来查看重建进度：
```
.\storcli64.exe /c0/e252/sall show rebuild
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = Show Drive Rebuild Status Succeeded.


----------------------------------------------------------
Drive-ID    Progress% Status          Estimated Time Left
----------------------------------------------------------
/c0/e252/s3 -         Not in progress -
/c0/e252/s4 -         Not in progress -
/c0/e252/s5 -         Not in progress -
/c0/e252/s6 -         Not in progress -
/c0/e252/s7 9         In progress     49 Minutes
----------------------------------------------------------

#这里我直接使用的是/c0/e252/sall查看所有硬盘的重建进度，也可以直接查看/c0/e252/s7的重建进度。
```
可以看到slot7盘现在重建进度为9%，预估剩余时间为49分钟，这里剩余时间不是很准，只能作为参考。

```
storcli /cx[/ex]/sx pause rebuild
#暂停当前重建的硬盘

storcli /cx[/ex]/sx resume rebuild
#恢复暂停状态的硬盘，使其重新进入重建状态

storcli /cx[/ex]/sx show rebuild
#显示重建进度

storcli /cx[/ex]/sx start rebuild
#开始重建

storcli /cx[/ex]/sx stop rebuild
#停止重建
```

# 点亮硬盘
可以在系统下定位驱动器并激活物理磁盘活动LED，使现场工程师更方便的确认出现故障需要更换的硬盘槽位。
storcli /cx[/ex]/sx start locate
storcli /cx[/ex]/sx stop locate
```
.\storcli64.exe /c0/e252/s3 start locate
#点亮slot3槽位的故障灯

.\storcli64.exe /c0/e252/s3 stop locate
#关闭slot3槽位的故障灯
```

# 关闭阵列卡告警
```
storcli /cx set alarm=<on|off|silence>
- on：打开阵列卡蜂鸣器
- off： 关闭阵列卡蜂鸣器
- silence： 暂时关闭阵列卡蜂鸣器，可以理解为关闭本次告警
```

# 查询超级电容（BBU）的相关信息
```
.\storcli64.exe /c0/cv show all
CLI Version = 007.1017.0000.0000 May 10, 2019
Operating system = Windows 10
Controller = 0
Status = Success
Description = None


Cachevault_Info :
===============

--------------------
Property    Value
--------------------
Type        CVPM02
Temperature 31 C
State       Optimal
--------------------


Firmware_Status :
===============

---------------------------------------
Property                         Value
---------------------------------------
Replacement required             No
No space to cache offload        No
Module microcode update required No
---------------------------------------


GasGaugeStatus :
==============

------------------------------
Property                Value
------------------------------
Pack Energy             302 J
Capacitance             101 %
Remaining Reserve Space 93
------------------------------


Design_Info :
===========

------------------------------------
Property                 Value
------------------------------------
Date of Manufacture      21/12/2014
Serial Number            34981
Manufacture Name         LSI
Design Capacity          283 J
Device Name              CVPM02
tmmFru                   N/A
CacheVault Flash Size    N/A
tmmBatversionNo          0x0
tmmSerialNo              0x36c5
tmm Date of Manufacture  22/04/2015
tmmPcbAssmNo             L22541903A
tmmPCBversionNo          0x03
tmmBatPackAssmNo         49571-03A
scapBatversionNo         0x0
scapSerialNo             0x88a5
scap Date of Manufacture 21/12/2014
scapPcbAssmNo            1700054483
scapPCBversionNo          K
scapBatPackAssmNo        49571-03A
Module Version           25849-01
------------------------------------


Properties :
==========

--------------------------------------------------------------
Property             Value
--------------------------------------------------------------
Auto Learn Period    27d (2412000 seconds)
Next Learn time      2019/09/05  17:04:28 (621018268 seconds)
Learn Delay Interval 0 hour(s)
Auto-Learn Mode      Transparent
--------------------------------------------------------------
```
这里当回显信息的“State”显示为“FAILED”时，需要更换超级电容（BBU）。