---
title: Ubuntu使用SS/SSR并开启透明代理
date: 2019-05-15 21:50:10
tags: [Ubuntu,SS,SSR,Global proxy,UDP proxy,python-ssr,socks proxy]
categories: Ubuntu
---
<center>
ss-redir 是 ss-libev、ssr-libev 中的一个工具<br/>
配合 iptables 可以在 Linux 上实现 ss、ssr 透明代理<br/>
ss-redir 的 TCP 透明代理是通过 REDIRECT 方式实现的<br/>
而 UDP 透明代理是通过 TPROXY 方式实现的<br/>
强调一点<br/>
利用 ss-redir 实现透明代理必须使用 ss-libev 或 ssr-libev<br/>
而python、go 等版本没有 ss-redir、ss-tunnel 程序
</center>
<!--more-->

## ss/ssr透明代理
当然，ss、ssr 透明代理并不是只能用 ss-redir 来实现，使用 ss-local + redsocks2/tun2socks 同样可以实现 socks5（ss-local 是 socks5 服务器）全局透明代理；ss-local + redsocks2 实际上是 ss-redir 的分体实现，TCP 使用 REDIRECT 方式，UDP 使用 TPROXY 方式；ss-local + tun2socks 则相当于 Android 版 SS/SSR 的 VPN 模式，因为它实际上是通过一张虚拟的 tun 网卡来进行代理的。
最后说一下v2ray的透明代理，其实原理和 ss/ssr-libev 一样，v2ray 可以看作是 ss-local、ss-redir、ss-tunnel、ss-server 四者的合体，因为同一个 v2ray 程序既可以作为 server 端，也可以作为 client 端。所以 v2ray 的透明代理也有两种实现方式，一是利用对应的 ss-redir + iptables，二是利用对应的 ss-local + redsocks2/tun2socks（redsocks2/tun2socks 可以与任意 socks5 代理组合，实现透明代理）。

## ss-tproxy
这里我们使用ss-tproxy和redsocks2配合使用，来实现代理转换
关于ss-tproxy，可以查看下面的链接。
[请点击查看](https://www.zfl9.com/ss-redir.html 'ss-tproxy')

## 安装配置ssr
这里我使用的是shell脚本封装过的ssr，原版本是python实现的
### 1、下载ssr install.sh
`wget https://raw.githubusercontent.com/the0demiurge/CharlesScripts/master/charles/bin/ssr`
### 2、运行ssr进行初始化
`bash ssr install`
### 3、配置ssr相关参数，自动启动ssr监听端口
```
ssr config

"local_address": "0.0.0.0",
"local_port": 61080,
#这里需要注意的是local_port后面需要用到，我这里设置的是61080
#如果你不理解你在干什么，请将local_port设置为和我相同
#下面//的两处注释也要删除
#保存后，如果输出有你的ssr节点信息则表示配置成功
#否则请检测配置文件或节点
```
### 4、ssr脚本使用方法
```
ssr start      #启动ssr
ssr restart    #重启ssr
ssr stop       #停止ssr
ssr help       #查看帮助
ssr uninstall  #卸载ssr
```

## 安装配置redsocks2
### 1、下载redsocks2源码
```
git clone https://github.com/semigodking/redsocks/
cd redsocks
git checkout 53cad23  #这一步是必须的，因为最新版本已经放弃对OpenSSL-ss的支持

```
### 2、编译安装redsocks2
如果你不确定具有哪个版本，并且是Debian系列，请尝试OpenSSL安装方式
#### 如果要在使用OpenSSL 1.1.0+的系统上进行编译：
```
git apply patches/disable-ss.patch
make -j4
```
#### 如果安装的是PolarSSL，使用下面的命令编译
```
make USE_CRYPTO_POLARSSL = true
make -j4
```
### 3、配置redsocks2(假设路径为/etc/redsocks2.conf)
```
vim /etc/redsocks2.conf
#这是一个由你创建的空文件，请写入下面内容
base {
  daemon = on;
  log_info = on;
  reuseport = on;
  redirector = iptables;
  log = "file:/var/log/redsocks2.log";
}

redsocks {
  local_ip = 0.0.0.0;
  local_port = 60080;
  ip = 127.0.0.1;
  port = 61080;
  type = socks5;
}

redudp {
  local_ip = 0.0.0.0;
  local_port = 60080;
  ip = 127.0.0.1;
  port = 61080;
  type = socks5;
}
#这里的port是上面ssr config配置中的local_port
#如果你按照我的方式进行配置，那边不需要修改任何地方
#请自信的键入:wq
```

## 安装配置ss-tproxy
### 1、安装脚本
#### 安装方法
```
git clone https://github.com/zfl9/ss-tproxy
cd ss-tproxy
cp -af ss-tproxy /usr/local/bin
chmod 0755 /usr/local/bin/ss-tproxy
chown root:root /usr/local/bin/ss-tproxy
mkdir -m 0755 -p /etc/ss-tproxy
cp -af ss-tproxy.conf gfwlist.* chnroute.* /etc/ss-tproxy
chmod 0644 /etc/ss-tproxy/* && chown -R root:root /etc/ss-tproxy
```

#### 附上删除脚本的方法
```
ss-tproxy stop
ss-tproxy flush-iptables
rm -fr /etc/ss-tproxy /usr/local/bin/ss-tproxy
```

#### 附上脚本使用方法
```
ss-tproxy help：查看帮助
ss-tproxy start：启动代理
ss-tproxy stop：关闭代理
ss-tproxy restart：重启代理
ss-tproxy status：代理状态
ss-tproxy check-command：检查命令是否存在
ss-tproxy flush-dnscache：清空 DNS 查询缓存
ss-tproxy flush-gfwlist：清空 ipset-gfwlist IP 列表
ss-tproxy update-gfwlist：更新 gfwlist（restart 生效）
ss-tproxy update-chnonly：更新 chnonly（restart 生效）
ss-tproxy update-chnroute：更新 chnroute（restart 生效）
ss-tproxy show-iptables：查看 iptables 的 mangle、nat 表
ss-tproxy flush-iptables：清空 raw、mangle、nat、filter 表
```
### 2、配置脚本
```
vim /etc/ss-tproxy/ss-tproxy.conf
#取消mode='global'的注释，
#将mode='chnroute'注释掉
#在proxy项中改为下面的代码

proxy_tproxy='false'
proxy_server=(1.2.3.4)
proxy_tcport='60080'  #请不要尝试修改tcp端口，除非你知道它的作用
proxy_udport='60080'  #请不要尝试修改udp端口，除非你知道它的作用
proxy_runcmd='start_ssrlocal_redsocks2'
proxy_kilcmd='bash /root/ssr stop && kill -9 $(pidof redsocks2)'
#这里ssr的路径根据自己的安装目录确定
#然后在 ss-tproxy.conf 的任意位置定义我们的start_ssrlocal_redsocks2函数
#比如在文件末尾添加如下代码：

function start_ssrlocal_redsocks2 {
    bash /root/ssr start
    /usr/local/bin/redsocks2 -c /etc/redsocks2.conf
}

```
### 3、运行脚本
```
ss-tproxy start
#如果输出没有报错
#运行下面命令来检查是否代理成功
curl ip.cip.cc
```

文章比较长，自己当时记录的文档比较乱，是结合记忆来写的，如果有不对的地方请留言
感谢你们的支持哦!

**end**
