---
title: Ubuntu18.04修改DNS 解决resolvconf-pull-resolved.service服务启动失败
date: 2019-06-18 22:04:08
tags: [Ubuntu,Ubuntu18.04,Linux,resolv]
categories: Ubuntu
---

<center>
今天在给Ubuntu18配置静态IP时遇到了DNS解析错误<br/>
并在重启后遇到了resolvconf-pull-resolved.service服务启动失败<br/>
在通过简单的修改配置后解决了这个问题
</center>
<!--more-->

## Ubuntu18.04修改DNS
现在很多网上的教程给出修改DNS的方法无非就是修改resolv.conf
修改/etc/systemd/resolved.conf
但是这文件修改不仅不能使DNS恢复正常，还会造成resolvconf-pull-resolved.service服务的异常
还有修改/etc/resolvconf/resolv.conf.d/base  head tail等，我测试过都是不可行的，也可能在其他linux中可以，但至少我尝试后是不可以的。

下面我给出一个我在Ubuntu18.04（kernel 4.20）上成功修改DNS的方法：
```
rm /etc/resolv.conf
ln -s /run/resolvconf/resolv.conf /etc/resolv.conf
resolvconf -u
#使修改生效
cat /etc/resolv.conf
#现在resolv.conf已经变成了/run/resolvconf/resolv.conf中的内容，因为我们设置了一个软链接。
```

## resolvconf-pull-resolved.service服务启动失败
其实这个问题的解决也非常简单，按上面的修改DNS的方法代码走一遍即可。
