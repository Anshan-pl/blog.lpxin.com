---
title: 配置Redhat8 PXE服务器安装系统
date: 2019-09-22 20:20:58
tags: [Redhat,Centos,Linux,PXE,UEFI,BIOS]
categories: PXE
---

<center>
通过配置PXE服务器<br/>
通过网络引导安装系统<br/>
包括但不限于目前主流的Linux和Windows操作系统<br/>
可基于BIOS或UEFI
</center>
<!--more-->

# 先决条件
PXE引导服务器需要DHCP，TFTP、FTP、samba等不同的服务才能正常运行。尽管不必在一台计算机上配置所有这些服务，并且如果您具有正在运行的FTP服务器，则可以将所需的安装文件放在现有的FTP服务器上。同样，如果您为网络配置了DHCP服务器，则可以在其中定义DHCP配置。

但是，为简单起见，这里将在同一台计算机上配置所有这些服务。


# 配置PXE网络
我这里的测试环境使用的是VMware虚拟机，添加了一个主机模式网卡，并把默认的DHCPD服务关闭了。
```
[root@localhost ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens38
TYPE=Ethernet
BOOTPROTO=static
IPADDR=10.0.0.1
NETMASK=255.255.255.0
NAME=ens38
DEVICE=ens38
ONBOOT=yes

#配置完成后重启ens38接口，请以实际环境配置网络
[root@localhost ~]# nmcli connection reload
[root@localhost ~]# nmcli d connect ens38
Device 'ens38' successfully activated with 'be9e2b6b-674b-771d-7251-f3b49b3d23e0'.
```
如果这里你在启动网卡时遇到了错误，请查看配置文件是否添加了NM_CONTROLLED="no"参数，将其改为yes或删除该配置项！

# 配置DHCP


## 安装dhcp-server软件包
```
[root@localhost ~]# yum install dhcp -y
***************省略N行*****************
Installed:
  dhcp.x86_64 12:4.2.5-77.el7.centos

Complete!
```

## 配置DHCP服务
```
[root@localhost ~]# vi /etc/dhcp/dhcpd.conf


#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#

option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

subnet 10.0.0.0 netmask 255.255.255.0 {
	option routers 10.0.0.254;
	range 10.0.0.2 10.0.0.253;

	class "pxeclients" {
	  match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
	  next-server 10.0.0.1;

	  if option architecture-type = 00:07 {
	    filename "shim.efi";
	  } else {
	    filename "pxelinux.0";
		}
  }
}
```

DHCP服务配置直接参考了redhat7官方的安装指南，[可以点击跳转查看](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/chap-installation-server-setup)

## 启动并启用DHCP服务
启动并启用dhcpd.service。
```
[root@localhost ~]# systemctl start dhcpd.service && systemctl enable dhcpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/dhcpd.service to /usr/lib/systemd/system/dhcpd.service.
```

允许通过Redhat防火墙的DHCP服务，另外，proxy-dhcp端口是传播TFTP服务器IP地址所必需的！
```
[root@localhost ~]# firewall-cmd --permanent --add-service={dhcp,proxy-dhcp}
success
[root@localhost ~]# firewall-cmd --reload
success
```
注意，这里我们在配置防火墙策略时加上了--permanent参数，该参数将使配置永久保存，加上该参数后防火墙配置不是立即生效的，所以需要使用firewall-cmd --reload命令使其重新读取配置文件。
嫌配置防火墙麻烦的可以直接关闭防火墙`systemctl stop firewalld`

# 配置TFTP服务
使用yum安装TFTP（Trivial FTP）服务器。
```
[root@localhost ~]# yum install tftp-server -y
```
## 启动并启用TFTP服务
```
[root@localhost ~]# systemctl start tftp.service && systemctl enable tftp.service
Created symlink from /etc/systemd/system/sockets.target.wants/tftp.socket to /usr/lib/systemd/system/tftp.socket.

[root@localhost ~]# firewall-cmd --permanent --add-service=tftp
success
[root@localhost ~]# firewall-cmd --reload
success
```

# 配置FTP服务
需要FTP服务才能将OS安装介质共享到PXE引导客户端。一些系统管理员使用NFS代替FTP服务。但是，如果我们使用NFS，则当我们要将Microsoft Windows OS安装选项添加到我们的PXE引导服务器时，也必须配置Samba。因此，最好使用一种通用技术来共享用于不同操作系统的OS安装介质。HTTP也是FTP的良好替代，可用于共享OS安装介质。

## 安装VSFTPD服务器
```
[root@localhost ~]# yum install vsftpd -y
```

## 启动并启用VSFTPD服务
```
[root@localhost ~]# systemctl enable vsftpd.service && systemctl start vsftpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/vsftpd.service to /usr/lib/systemd/system/vsftpd.service.


[root@localhost ~]# firewall-cmd --permanent --add-service=ftp
success
[root@localhost ~]# firewall-cmd --reload
success
```

# 安装syslinux
Syslinux软件包提供了各种引导程序，包括FAT文件系统，可以引导到Microsoft Windows环境，但是请注意，syslinux提供的Windows引导程序无法在UEFI模式下使用！

```
[root@localhost ~]# yum install syslinux -y
```

# 配置PXE引导
将必要的引导程序（由syslinux提供）复制到/var/lib/tftpboot目录。
```
[root@localhost ~]# cp -v /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
'/usr/share/syslinux/pxelinux.0' -> '/var/lib/tftpboot/pxelinux.0'
[root@localhost ~]# cp -v /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
'/usr/share/syslinux/menu.c32' -> '/var/lib/tftpboot/menu.c32'
[root@localhost ~]# cp -v /usr/share/syslinux/mboot.c32 /var/lib/tftpboot/
'/usr/share/syslinux/mboot.c32' -> '/var/lib/tftpboot/mboot.c32'
[root@localhost ~]# cp -v /usr/share/syslinux/chain.c32 /var/lib/tftpboot/
'/usr/share/syslinux/chain.c32' -> '/var/lib/tftpboot/chain.c32'
```
创建必要的目录
```
[root@localhost ~]# mkdir /var/lib/tftpboot/pxelinux.cfg
[root@localhost ~]# mkdir -p /var/lib/tftpboot/networkboot/rhel7
[root@localhost ~]# mkdir /var/ftp/pub/rhel7
```

## 提前rhel7 ISO
将RHEL 7.7 ISO的内容复制到/var/ftp/pub/rhel7。
```
[root@localhost ~]# mkdir /mnt/iso
[root@localhost ~]# mount -t iso9660 /dev/cdrom /mnt/iso -o loop,ro
[root@localhost ~]# cp -rf /mnt/iso/* /var/ftp/pub/rhel7
[root@localhost ~]# chmod 777 -R /var/ftp/pub/rhel7
```
这里将ISO挂载为了只读模式，如果需要读写，请将-o loop,ro修改为-o loop,rw，但不建议这样做。
接下来将RHEL 7.7的启动映像复制到/var/lib/tftpboot/networkboot/rhel7目录。
```
[root@localhost ~]# cp /var/ftp/pub/rhel7/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot/networkboot/rhel7/
```

## 配置PXE BIOS模式下的启动菜单
配置PXE BIOS引导菜单，并在其中添加Red Hat Enterprise Linux 7安装选项。
```
[root@localhost ~]# vi /var/lib/tftpboot/pxelinux.cfg/default

default menu.c32
prompt 0
timeout 30
menu title AnShan PXE Menu
label Install RHEL 7.7
kernel /networkboot/rhel7/vmlinuz
append initrd=/networkboot/rhel7/initrd.img inst.repo=ftp://10.0.0.1/pub/rhel7
```
到这里我们创建了一个RHEL7的PXE网络引导安装项，将新系统连接到网络并启动。新系统将自动从我们的DHCP服务器获取IP地址，并从PXE服务器获取PXE引导菜单，由于到目前为止，我们只有一个安装选项，因此，请按<ENTER>键开始安装。下面将继续添加Windows10的安装项

# 添加Windows10安装候选项
## 安装并配置Samba服务
使用Samba服务器与PXE客户端共享MS Windows 10操作系统的安装媒体。
```
[root@localhost ~]# yum install samba -y
```
创建一个目录以共享Windows 10安装媒体。
`[root@localhost ~]# mkdir /smbdir`
## 调整SELinux权限
这里调整SELinux权限需要用到semanage命令，而我这里的redhat7默认没有安装，所以先安装该软件包，通过provides定位具体所在包
```
[root@localhost ~]# yum provides semanage
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
base/x86_64/filelists_db                                                              | 7.3 MB  00:00:00
extras/x86_64/filelists_db                                                            | 207 kB  00:00:00
updates/x86_64/filelists_db                                                           | 1.0 MB  00:00:00
policycoreutils-python-2.5-33.el7.x86_64 : SELinux policy core python utilities
Repo        : base
Matched from:
Filename    : /usr/sbin/semanage

[root@localhost ~]# yum install policycoreutils-python-2.5-33.el7.x86_64 -y
```
可以看到该命令属于policycoreutils-python-2.5-33.el7.x86_64包，使用install安装即可。
调整SELinux权限并使其立即生效：
```
[root@localhost ~]# semanage fcontext -a '/smbdir(/.*)?' -t samba_share_t
[root@localhost ~]# restorecon -Rv /smbdir
restorecon reset /smbdir context unconfined_u:object_r:default_t:s0->unconfined_u:object_r:samba_share_t:s0
```

## 创建Samba用户
```
[root@localhost ~]# useradd -s /sbin/nologin anshan
[root@localhost ~]# smbpasswd -a anshan
New SMB password:
Retype new SMB password:
Added user anshan.
```
我这里创建的用户为anshan，密码也同样是anshan，你不必与我的相同。
继续修改Samba共享目录的所有者
```
[root@localhost ~]# chown anshan:anshan /smbdir
```

## 防火墙允许Samba服务
```
[root@localhost ~]# firewall-cmd --permanent --add-service=samba
success
[root@localhost ~]# firewall-cmd --reload
success
```

## 配置Samba服务
```
[root@localhost ~]# vi /etc/samba/smb.conf
# See smb.conf.example for a more detailed config file or
# read the smb.conf manpage.
# Run 'testparm' to verify the config is correct after
# you modified it.

[install]
  comment = Installation Media
  path = /smbdir
  public = yes
  writable = no
  printable = no
  browseable = yes
```

## 启动并启用Samba
```
[root@localhost ~]# systemctl start smb nmb && systemctl enable smb nmb
Created symlink from /etc/systemd/system/multi-user.target.wants/smb.service to /usr/lib/systemd/system/smb.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/nmb.service to /usr/lib/systemd/system/nmb.service.
```

## 提取Windows10 ISO
附加MS Windows 10 ISO/DVD并将其安装在/mnt/iso（根据你的的选择，可以使用任何安装点）。
```
[root@localhost ~]# mount /dev/cdrom /mnt/iso -o loop,ro
[root@localhost ~]# mkdir /smbdir/win10
[root@localhost ~]# cp -rf /mnt/iso/* /smbdir/win10
[root@localhost ~]# chmod 777 -R /smbdir/win10
```
-t iso9660参数是非必需的。

## 创建自定义的Windows PE ISO
在本地环境下载ADK或者使用虚拟机都可以，但必须是windows系统，因为需要Windows ADK（评估和部署套件）来创建Windows PE iso。因此，我需要从[Microsoft](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install)的网站上下载了它并将其安装在Windows 10客户端上。
首先下载并安装Download the Windows PE add-on for the ADK（下载用于ADK的Windows PE加载项），
接着继续下载安装Download the Windows ADK for Windows 10, version 1903（下载适用于Windows 10版本1903的Windows ADK）。
请记住顺序，必须要先安装用于ADK的Windows PE加载项，接着再安装用于Windows 10版本的Windows ADK！
安装完成后在[开始]菜单中会存在一个部署和映像工具环境，右键使用管理员身份打开后是一个CLI类似cmd的命令终端，记住，这里需要使用管理员身份运行！
```
C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools>cd ..

C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit>dir
 驱动器 C 中的卷没有标签。
 卷的序列号是 406B-1A56

 C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit 的目录

2019/09/19  16:32    <DIR>          .
2019/09/19  16:32    <DIR>          ..
2019/09/19  16:31    <DIR>          Common
2019/09/19  16:31    <DIR>          Deployment Tools
2019/09/19  16:31    <DIR>          Docs
2019/09/19  16:31    <DIR>          Imaging and Configuration Designer
2019/09/19  16:32    <DIR>          User State Migration Tool
2019/09/19  15:45    <DIR>          Windows Preinstallation Environment
2019/09/19  16:32    <DIR>          Windows Setup
               0 个文件              0 字节
               9 个目录 35,926,810,624 可用字节

C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit>cd Windows Preinstallation Environment

C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment>copype amd64 c:\WinPE_amd64

===================================================
Creating Windows PE customization working directory

    c:\WinPE_amd64
===================================================
********************省略N行*************************
复制了 153 个文件
已复制         1 个文件。
已复制         1 个文件。
已复制         1 个文件。

Success

```
我们将自定义启动脚本startcmd.net，因此MS Windows 10安装程序将自动启动。因此，挂载映像文件并相应地对其进行自定义。
```
C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment>Dism /Mount-Image /ImageFile:"c:\WinPE_amd64\media\sources\boot.wim" /index:1 /MountDir:"c:\WinPE_amd64\mount"

部署映像服务和管理工具
版本: 10.0.18362.1

正在安装映像
[==========================100.0%==========================]
操作成功完成。

C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment>notepad c:\WinPE_amd64\mount\Windows\System32\Startnet.cmd

#现在，编辑startnet.cmd,在打开的文本文件中添加如下两行：
wpeinit
net use z: \\10.0.0.1\install\win10 /user:anshan anshan
z:\setup.exe
#修改完毕后保存退出。

#保存并卸载图像文件。
C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment>Dism /Unmount-Image /MountDir:"c:\WinPE_amd64\mount" /commit

部署映像服务和管理工具
版本: 10.0.18362.1

正在保存映像
[==========================100.0%==========================]
正在卸载映像
[==========================100.0%==========================]
操作成功完成。

#生成winpe.iso文件。
C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment>MakeWinPEMedia /ISO c:\WinPE_amd64 c:\winpe.iso
Creating c:\winpe.iso...

100% complete

Success

```
将winpe.iso文件上传到PXE服务器的Samab目录下，并将其复制到/var/lib/tftpboot/networkboot/win10/目录即可。

## 将MS Windows 10安装选项添加到PXE引导菜单
在tftpboot目录中复制MS Windows10的内核启动映像。
```
[root@localhost ~]# cp /usr/share/syslinux/memdisk /var/lib/tftpboot/
```
注意，我这里的memdisk引导文件位于/usr/share/syslinux/memdisk，可能有些系统在/usr/lib/syslinux/memdisk，不确定的可以使用find命令查找：
```
[root@localhost ~]# find / -name memdisk
/usr/share/syslinux/memdisk
```
编辑基于BIOS的客户端的PXE引导菜单，添加MS Windows10启动项,在文件末尾添加以下菜单选项:
```
label Install MS Windows 10
menu label Install MS Windows 10
kernel memdisk
initrd /networkboot/windows10/winpe.iso
append iso raw
```
将新客户端连接到您的网络并打开它。它应该从DHCP服务器获取IP地址，并显示我们的PXE启动菜单，现在该PXE BIOS菜单下应该分别具有redhat和windows10的安装引导项。
注意，这里windows10PE安装程序的语言是英语，选择区域和语言的时候默认选择的是中国，待安装结束之后重启进入系统就正常是中午了。


# 配置PXE引导服务器以支持基于UEFI的系统
BIOS（基本输入/输出系统）和UEFI（统一可扩展固件接口）是计算机系统的两个固件接口，它们充当操作系统和计算机固件之间的解释器。两者均在制造时安装，并且是在打开计算机电源时运行的第一个程序。BIOS使用主引导记录（MBR）保存有关硬盘数据的信息，而UEFI使用GUID分区表（GPT）。
MBR在其表中使用32位条目，该条目将总物理分区限制为仅4个，每个分区的最大大小为2 TB。而GPT在其表中使用64位条目，这使其可以使用4个以上具有更大大小的物理分区。

## 配置DHCP服务以支持UEFI
在最开始配置DHCP的时候就已经添加上了对于UEFI模式的支持：
```
if option architecture-type = 00:07 {
  filename "shim.efi";
  } else {
  filename "pxelinux.0";
  }
```
这里00:07是固件发送的体系结构类型参数其代表传统的BIOS模式，而00:09则是UEFI模式，该参数由architecture-type来接收

## 配置PXE UEFI模式菜单
现在，我们需要一个引导程序（例如shim.efi）来支持UEFI客户端。该引导程序在Redhat 7.7 ISO中可用。
将shim.efi和grubx64.efi复制到/var/lib/tftpboot目录。
```
[root@localhost ~]# cp /boot/efi/EFI/redhat/shim.efi /var/lib/tftpboot/
[root@localhost ~]# cp /boot/efi/EFI/redhat/grubx64.efi /var/lib/tftpboot/
```
一般shim.efi文件在安装镜像ISO中的/boot目录都可以找到，我这里因为PXE环境就为rhel7，所以直接复制的是该系统/boot目录中的shim.efi文件。
我们的PXE BIOS菜单不适用于UEFI系统，因此我们必须为UEFI客户端创建另一个菜单。
菜单文件名为grub.cfg，位于/var/lib/tftpboot。因此，我们将在此文件中定义RHEL 7.7安装选项，如下所示：
```
[root@localhost ~]# vi /var/lib/tftpboot/grub.cfg
set timeout=60
menuentry 'Install RHEL 7.7' {
  linuxefi /networkboot/rhel7/vmlinuz inst.repo=ftp://10.0.0.1/pub/rhel7/
  initrdefi /networkboot/rhel7/initrd.img
}
```
重新启动tftp.service以应用更改。
`[root@localhost ~]# systemctl restart tftp`
UEFI配置已完成,要测试配置，请将基于UEFI的系统连接到网络，然后将其打开。
如果你遇到了类似time out或者被拒绝之类的问题，不要犹豫，请使用chmod 777 -R对其添加权限。
