---
title: 网络设备CLI基础命令
date: 2019-09-26 20:28:53
tags: [Huawei,Cisco,ARP,IOS,eNSP,Cisco Packet Tracer,eve-ng]
categories: XCNA
---

<center>
模拟器尝试路由器以及交换机命令<br/>
思科模拟器使用eve-ng<br/>
华为模拟器使用eNSP
</center>
<!--more-->

# 重置路由器
## Cisco
```
Router#erase startup-config 
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm]
[OK]
Erase of nvram: complete

Router#reload

System configuration has been modified. Save? [yes/no]: no
Proceed with reload? [confirm]

*Sep 26 17:33:17.823: %SYS-5-RELOAD: Reload requested by console. Reload Reason: Reload Command.

                ROM: reload requested...
```
注意，使用raload命令使路由器重新加载后，在模拟器环境下设备会关机，生产环境未测试。

## Huawei
```
<Huawei>reset saved-configuration
This will delete the configuration in the flash memory.

The device configuratio
ns will be erased to reconfigure.

Are you sure? (y/n)[n]:y
 The config file does not exist.
#键入y确定删除配置文件


<Huawei>reboot
Info: The system is comparing the configuration, please wait.
Warning: All the configuration will be saved to the next startup configuration. 
Continue ? [y/n]:n
System will reboot! Continue ? [y/n]:y
Info: system is rebooting ,please wait...
#第一个交互键入n不保存配置
#第二个交互键入y确定重启
```

# Telnet
## Cisco
### 使用用户名以及密码登陆
```
Router>enable

Router#configure terminal

Router(config)#username admin password admin
#设置用户名和密码均为admin

Router(config)#line vty ?   
  <0-1869>  First Line number

Router(config)#line vty 0 1869
#最大允许同时开启1869个会话

Router(config-line)#login local
#开启本地登陆

Router(config)#enable password admin
#配置enable密码，否则使用telnet登陆的用户无法enable
```

### 直接使用密码登陆
```
Router>enable

Router#configure terminal

Router(config)#line vty 0 1869

Router(config-line)#password admin
#注意，直接配置密码密码只有在(config-line)模式下可用

Router(config-line)#login
#默认login就是开启的
Router(config)#enable password admin
```

### 查看密码
```
Router#show run | section password
no service password-encryption
enable password admin
username admin password 0 admin
 password admin
 password admin
```
cisco默认是对密码不加密的，所以可以直接使用show run命令来查看密码，而华为默认是加密的，cisco上可以使用service password-encryption命令开启密码保护，如下：
```
Router(config)#service password-encryption

Router#show run | section password
service password-encryption
enable password 7 1218011A1B05
username admin password 7 045A0F0B062F
 password 7 082048430017
 password 7 082048430017
```

## Huawei
### 使用用户名以及密码登陆
```
<Huawei>system-view 
Enter system view, return user view with Ctrl+Z.

[Huawei]aaa
#进入AAA界面

[Huawei-aaa]local-user admin password cipher admin
#设置AAA用户名及密码

[Huawei-aaa]local-user admin service-type telnet
#设置telnet允许admin用户登陆

[Huawei-aaa]local-user admin privilege level 15
#设置admin用户登陆后用户等级15，否则使用admin登陆后无法使用system-view进入特权模式

[Huawei-aaa]quit

[Huawei]user-interface vty ?
  INTEGER<0-4,16-20>  The first user terminal interface to be configured

[Huawei]user-interface vty 0 4
#最大同时允许4个会话

[Huawei-ui-vty0-4]authentication-mode aaa
#启用AAA认证验证
```

### 直接使用密码登陆
```
<Huawei>system-view

[Huawei]user-interface vty 0 4

[Huawei-ui-vty0-4]authentication-mode password
Please configure the login password (maximum length 16):admin

[Huawei-ui-vty0-4]set authentication password cipher admin123
#该命令为修改密码

[Huawei-ui-vty0-4]user privilege level 15
#设置用户等级为15
```

# SSH
## Cisco
```
Router>enable

Router#configure terminal 
Enter configuration commands, one per line.  End with CNTL/Z.


Router(config)#hostname R1

R1(config)#ip domain-name r1.cisco.com
#配置R1的域名为r1.cisco.com

R1(config)#crypto key generate rsa
The name for the keys will be: R1.r1.cisco.com
Choose the size of the key modulus in the range of 360 to 4096 for your
  General Purpose Keys. Choosing a key modulus greater than 512 may take
  a few minutes.

How many bits in the modulus [512]: 1024
% Generating 1024 bit RSA keys, keys will be non-exportable...
[OK] (elapsed time was 2 seconds)
#生成RSA加密的随机数

R1(config)#username admin privilege 15 password admin

R1(config)#line vty 0 1869

R1(config-line)#login local

R1(config-line)#transport input ssh
#只允许ssh协议连接
```

## Huawei
```
<AR1>system-view

[AR1]rsa local-key-pair create 
The key name will be: Host
% RSA keys defined for Host already exist.
Confirm to replace them? (y/n)[n]:y
The range of public key size is (512 ~ 2048).
NOTES: If the key modulus is greater than 512,
       It will take a few minutes.
Input the bits in the modulus[default = 512]:1024
Generating keys...
......................................................++++++
............++++++
.........++++++++
..++++++++

[AR1-aaa]local-user admin privilege level 15 password cipher admin

[AR1-aaa]local-user admin service-type ssh

[AR1]ssh user admin authentication-type all
 Authentication type setted, and will be in effect next time
#设置ssh的admin用户验证类型为所有

[AR1-ui-vty0-4]authentication-mode aaa
#允许AAA用户登陆

[AR1-ui-vty0-4]protocol inbound ssh
#只允许ssh协议登陆

[AR1]stelnet server enable
Info: Succeeded in starting the STELNET server.
#开机server端的stelnet服务
```
以上是华为设备Server端开启方法，而在Client端，也需要开启ssh客户端，如下：
```
[AR1]ssh client first-time enable
#开启客户端的ssh，即stelnet命令

[AR2]stelnet 12.1.1.1
Please input the username:admin
Trying 12.1.1.1 ...
Press CTRL+K to abort
Connected to 12.1.1.1 ...
The server is not authenticated. Continue to access it? (y/n)[n]:y
Sep 27 2019 17:56:07-08:00 AR2 %%01SSH/4/CONTINUE_KEYEXCHANGE(l)[0]:The server h
ad not been authenticated in the process of exchanging keys. When deciding wheth
er to continue, the user chose Y. 
[AR2]
Save the server's public key? (y/n)[n]:y
The server's public key will be saved with the name 12.1.1.1. Please wait...

Sep 27 2019 17:56:09-08:00 AR2 %%01SSH/4/SAVE_PUBLICKEY(l)[1]:When deciding whet
her to save the server's public key 12.1.1.1, the user chose Y.
[AR2]
Enter password:
<AR1>
```
