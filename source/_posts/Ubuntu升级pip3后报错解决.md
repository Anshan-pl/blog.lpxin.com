---
title: Ubuntu升级pip3后报错解决
date: 2019-05-15 20:28:30
tags: [Ubuntu,python3,pip3]
categories: Ubuntu
---
<center>
Traceback (most recent call last): <br/>
  File "/usr/bin/pip3", line 9, in <module> <br/>
    from pip import main <br/>
ImportError: cannot import name 'main'
</center>
<!--more-->

如果你遇到了和我一样的报错，那么恭喜你
解决方法非常简单！
```
sudo vim /usr/bin/pip3

#注意：由于若/usr/bin/pip3是只读文件，不加sudo ，可能会提示权限不足，若在只读权限下强制保存会导致文件受损，建议修改配置文件时先查看是否具有权限 
```
修改内容如下：
原文：`from pip import main`
修改后：`from pip._internal import main`

提示：普通用户身份升级pip3之后，使用root用户可能还在原版本，可以把pip3配置文件改回去，升级pip3，之后再修改为from pip._internal import main

**end**
