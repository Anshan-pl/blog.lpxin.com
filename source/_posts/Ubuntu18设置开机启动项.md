---
title: Ubuntu18设置开机启动项
date: 2019-05-16 15:31:09
tags: [Ubuntu,rc.local,systemctl]
categories: Ubuntu
---
<center>
Ubuntu 18不能再像Ubuntu 16一样通过编辑rc.local来设置开机启动脚本<br/>
通过简单设置后，可以使rc.local重新发挥作用
</center>
<!--more-->

## 1、建立rc-local.service文件
`sudo vim /etc/systemd/system/rc-local.service`

## 2、配置开机自启服务
将下列内容复制进rc-local.service文件
```
[Unit]
Description=/etc/rc.local Compatibility
ConditionPathExists=/etc/rc.local
 
[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99
 
[Install]
WantedBy=multi-user.target
```

## 3、创建rc.local
```
sudo vim /etc/rc.local

#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.
echo "看到这行字，说明添加自启动脚本成功。" >> /usr/local/test.log
exit 0
```

## 4、给rc.local赋权
`sudo chmod +x /etc/rc.local`

## 5、启动服务并检测状态
```
sudo systemctl enable rc-local
sudo systemctl start rc-local.service
sudo systemctl status rc-local.service
```
## 7、重启并检测test.log文件
`cat /usr/local/test.log`
这里你应该看到两行字，这是因为在重启前启动rc.local服务时也执行了一次/etc/rc.local文件。

如果你遇到了什么问题，或者文章里有不对的地方欢迎留言。
我会随时查看！

**end**
