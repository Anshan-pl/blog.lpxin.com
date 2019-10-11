---
title: Linux下使用Caddy代替Nginx，并配置php-fpm
date: 2019-05-16 18:02:00
tags: [Linux,Ubuntu,Centos,Caddy,Nginx,php-frp,https,Let’s Encrypt]
categories: Linux
---
<center>
Caddy是一个通用的HTTP/2 Web服务器，默认情况下提供HTTPS<br/>
并且Caddy有个最实用的功能，自动支持HTTPS<br/>
同样是使用 Let's Encrypt，换成 Nginx 我们就必须手工操作<br/>
并且还需要设置三个月更新证书的计划任务<br/>
而Caddy无声的完成了所有操作<br/>
并且支持HTTP/2，另外他的配置文件相较于Nginx也更为简单
</center>

![caddy](/images/post/caddy.png "caddy")
<!--more-->

## Caddy安装
```
curl https://getcaddy.com | bash -s personal
mkdir /etc/caddy
chown -R root:www-data /etc/caddy
mkdir /etc/caddy/conf
echo 'import ./conf/*' >> /etc/caddy/Caddyfile  #这条语句是指/etc/caddy/conf/路径下的配置文件都会被包含到Caddyfile中，有利于多站点的配置
mkdir /etc/ssl/caddy
chown -R www-data:root /etc/ssl/caddy
chmod 0770 /etc/ssl/caddy
mkdir /var/www
chown www-data:www-data /var/www

```

## 配置Caddy服务使其开机自启
```
sudo curl -s https://raw.githubusercontent.com/mholt/caddy/master/dist/init/linux-systemd/caddy.service -o /etc/systemd/system/caddy.service
sudo systemctl daemon-reload
sudo systemctl enable caddy
sudo systemctl start caddy
```
## 配置Caddyfile
Caddyfile是Caddy 的配置文件，刚才我们写入了一条 import ./conf/* 命令，表示将/etc/caddy/conf/下的所有文件都导入到配置文件中。[更多关于Caddyfile的配置方法请点击这里查看](https://caddyserver.com/docs "caddyfile")
```
#这里我给出一个最简单了例子
blog.lpxin.com {
  gzip  #表示启用gzip压缩，这能够有效地节约带宽，并提高响应速度。
  root /var/www/blog.lpxin.com  #和Nginx一样，Caddy也有一个root配置。静态资源的访问只要设置到对应的root即可。
}
```

## 配置Caddy使用php-fpm

### 安装并配置php-fpm
```
apt install php-fpm curl -y  #我这里目前的php-fpm版本是7.2
vim /etc/php/7.2/fpm/pool.d/www.conf  #根据你的php-fpm版本来修改路径
#取消下面的注释。
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
#保存并退出。

```

### 启动php-fpm并配置其随系统启动
```
systemctl start php7.2-fpm
systemctl enable php7.2-fpm

netstat -pl | grep php
#检查PHP-FPM套接字文件进程是否存在。
```

### Caddyfile添加php-fpm配置
修改刚刚Caddyfile的配置文件
```
blog.lpxin.com {
  gzip
  root /var/www/blog.lpxin.com
  fastcgi / /run/php/php7.2-fpm.sock php {  #注意，这里php-fpm套接字的路径请根据你的环境修改
    ext .php
    split .php
    index index.php
  }
}
```

## 重启Caddy并查看php-fpm配置是否成功
```
systemctl restart caddy
systemctl restart php7.2-fpm
echo '<?php phpinfo(); ?>' > /var/www/blog.lpxin.com/info.php
```
现在，访问你的域名，查看info.php文件是否可以打印出phpinfo！

## php-fpm扩展安装
比如的php程序可能需要php-curl、php-mbstring之类的扩展
那么你可以使用下面的命令进行简单的安装
`apt install php-curl php-mbstring -y`
OK，扩展已经安装成功了，就是这么简单。

**end**


