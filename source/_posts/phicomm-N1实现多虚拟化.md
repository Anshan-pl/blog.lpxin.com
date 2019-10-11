---
title: arm架构实现多虚拟化
date: 2019-05-19 09:20:23
tags: [phicomm,N1,虚拟化，QEMU,LXD,LXC,KVM]
categories: QEMU
---
<center>
之前有很多博纳云的用户找到我，说要参加N1镜像的内测<br/>
还和我抱怨没有留下联系的QQ或者群之类的信息<br/>
当时写那篇关于QEMU-KVM的时候没有想这么多，只是单纯的提了一下<br/>
但是看到有人响应，那么我这里就写一下arm框架KVM多虚拟化的实现~<br/>
这里提一下这种方法除了可以多挂以外，而且没有掉盘的风险哦！
</center>

![arm-kvm](/images/post/arm-kvm.jpg "arm-kvm")
<!--more-->

## 镜像选择
在开篇的地方，我说明一下，如果你觉得实现较为麻烦，那么请你直接拉到文章末尾，我已经做好了镜像和脚本，你可以加群找到哦~


这里我是用的phicomm N1
这里的镜像我们选择5.82版本，内核版本5.1，当然，我们可以自己编译最新的内核来使用，但是我可以很负责任的告诉你，5.82的镜像，已经够稳了，足够我们的测试喽
[放出大佬的镜像地址，请先下载镜像](https://yadi.sk/d/pHxaRAs-tZiei/5.82/S9xx/default "armbian-n1")
镜像名称选择Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img.xz
[呃，再直接给出下载链接吧，点击下载](https://s240myt.storage.yandex.net/rdisk/e5835fc69b1890d1e9bd2d529bf52283e1cead6ae40a83f799186bb87c25e0c8/5ce0eb7d/Lb2ke34_zZnnnWClJXZlU2IjqD0lThfzeqb7xeivGSJqhyS4m_GHfNU7ZoDeRMOVEnbZLloyzSRG_yplVgakcw==?uid=0&filename=Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img.xz&disposition=attachment&hash=xnxCtCUe//Abo40RhGRfigEhXq3PF70MFMIW3gEH0x8%3D%3A/5.82/S9xx/default/Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img.xz&limit=0&content_type=application%2Fx-xz&fsize=319956260&hid=b56f2a56d5a0f828142c72ce0dd7d5a6&media_type=unknown&tknv=v2&rtoken=o75zhzcETkvL&force_default=no&ycrid=na-09e9f9655b6a83491bbbbf2814acb4a1-downloader14e&ts=5893701441940&s=e0ddbe13e046453947cf1d57e2701d7d21f11616cadd6f07edfba192e051da4a&pb=U2FsdGVkX19khywtoTEPL2dsB0bo0hUdVxMcRT00TwRNd54y89S6ayb7IRS8rAzc6QpBCHZnSwVW52yte-w02F6nSesVZC46yO3SATYdPCg "armbian-n1-5.82")
或者使用wget命令
`wget https://s240myt.storage.yandex.net/rdisk/e5835fc69b1890d1e9bd2d529bf52283e1cead6ae40a83f799186bb87c25e0c8/5ce0eb7d/Lb2ke34_zZnnnWClJXZlU2IjqD0lThfzeqb7xeivGSJqhyS4m_GHfNU7ZoDeRMOVEnbZLloyzSRG_yplVgakcw==?uid=0&filename=Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img.xz&disposition=attachment&hash=xnxCtCUe//Abo40RhGRfigEhXq3PF70MFMIW3gEH0x8%3D%3A/5.82/S9xx/default/Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img.xz&limit=0&content_type=application%2Fx-xz&fsize=319956260&hid=b56f2a56d5a0f828142c72ce0dd7d5a6&media_type=unknown&tknv=v2&rtoken=o75zhzcETkvL&force_default=no&ycrid=na-09e9f9655b6a83491bbbbf2814acb4a1-downloader14e&ts=5893701441940&s=e0ddbe13e046453947cf1d57e2701d7d21f11616cadd6f07edfba192e051da4a&pb=U2FsdGVkX19khywtoTEPL2dsB0bo0hUdVxMcRT00TwRNd54y89S6ayb7IRS8rAzc6QpBCHZnSwVW52yte-w02F6nSesVZC46yO3SATYdPCg`

## 镜像安装
### 启动盘制作
这里我假设你的本地目录已经有了该固件，下载我们使用USB_Image_Tool这个工具来进行烧录，当然dd命令亦可
```
xz -d Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img.xz
sudo dd if=Armbian_5.82_Aml-s905_Ubuntu_bionic_default_5.1.0-rc1_20190427.img of=/dev/sdxx bs=4M
```
这里的 if= 和 of= 参数请自行修改

### 开机
到这里该怎么办。。。我相信不用我再重复写了，百度和Google上的教程没一千也有八百了，关键字 "N1 armbian"。因为这篇文章的主题不在于如何在N1上运行armbian，所以就不继续写了。

## QEMU磁盘文件制作
[这里，请你先看我的这篇文章，制作ubuntu-cloud-arm磁盘映像。](https://blog.lpxin.com/2019/05/16/QEMU%E8%B7%A8%E6%9E%B6%E6%9E%84%E4%BB%BF%E7%9C%9Faarch64%E4%BA%91%E6%9C%8D%E5%8A%A1/ "ubuntu-cloud-arm")
需要注意的地方是，这里我是使用x86框架来运行arm框架的ubuntu，你可以在x86的机器上部署QEMU环境，之后将制作完成的ubuntu-cloud导入到arm中运行。对的，你没有看错，在x86机器上修改好的磁盘文件是可以导入到arm机器上直接以相同的启动命令行来运行的，如果你的机器支持KVM，那么你可以在KVM模式下更好的运行虚拟机，在下面我会提到。

## KVM模式实现
接下来的步骤其实都已经在之前的文章里说过了，我这里只做一下整理，并给出需要注意的地方，具体请看这篇[QEMU进阶操作](https://blog.lpxin.com/2019/05/16/QEMU%E8%BF%9B%E9%98%B6%E6%93%8D%E4%BD%9C-%E4%BD%BF%E7%94%A8%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F-%E6%89%A9%E5%AE%B9%E9%95%9C%E5%83%8F-%E5%B9%B6%E5%88%9B%E5%BB%BA%E5%A2%9E%E9%87%8F%E9%95%9C%E5%83%8F/ "QEMU进阶")

### 启动桥接
这里我假设你已经有了可以使用的Ubuntu-cloud磁盘文件，桥接的方法在QEMU进阶操作第一栏就是“虚拟机网桥实现”。

### 扩展镜像容量
扩展方法在QEMU进阶操作第二栏就是“QEMU扩展镜像容量”。
提醒一下，如果你扩展的镜像容量大于2T（尽管我十分不建议你这样做），那么fdisk将无法进行操作，[你可以使用parted，请阅读这篇文章](https://blog.lpxin.com/2019/05/16/Linux%E4%BD%BF%E7%94%A8parted%E6%8C%82%E8%BD%BD%E5%A4%A7%E4%BA%8E2T%E7%A3%81%E7%9B%98/ "parted")

### 制作增量镜像
这一步不是必须的，但是制作增量镜像将为你省下很大的一笔硬盘开销，所以我认为它是必要的！
增量镜像请看QEMU进阶操作第四栏 “QEMU增量镜像制作”。

### 修改虚拟机mac地址
这一步如果你不是博纳云用户，那么你可能不需要，可以作为一个技巧来看。
修改mac地址，可能你想到的是这个命令
`ifconfig ethx hw ether yo:ur:ma:ca:dd:ress`
但是这里很不幸的告诉你，在虚拟机内部修改mac地址可能没有用，对于备份证书来运行博纳云的用户来说，它会起作用。

这里给出一个可用的方法，请看QEMU进阶操作第五栏 “修改虚拟机的MAC地址”。

### arm框架下运行QEMU-KVM
这里是最关键的地方，因为是否可以使用KVM模式将直接决定了你可以多开多少虚拟机。
关于如何使用，在QEMU进阶操作的最后一栏已经提到了。参数没什么需要改的，跟着文章里的运行即可
如果你是N1用户，那么你可以放心了，N1可以直接使用KVM模式，我已经测试了，树莓派3b以上版本应该也是可以的，可惜我手上没有树>
莓派的开发板无法进行实测。

这里只说一个注意点，对于博纳云用户，桥接是必须的，因为它是实现NAT1类型的前提，如果实现NAT1请看我博客的里的“PVE下通过Openwrt实现FULLCONE NAT（NAT1类型）” 对于ESXi实现，我也会写一篇教程。
我这里的桥接参数是
`ifname=tap0,script=no,downscript=no`
ifname=tapx 是你在创建tap设备时指定的名称，它可以无限多，我已经提到过了。
script=no,downscript=no，我这里的启动和关闭script全是空。请不要参照百度和Google上的各种脚本，他们很多都是错误，或者针对特定环境而在这里不可用的。如果你不知道你在做什么，那么请你不要修改！
至于为什么我不写脚本给大家，~~因为我比较懒~~，因为启动和关闭tap设备都较为简单，而且无需特殊的实现方法。

## 
到这里就已经结束了，你可以看到，很多地方都已经在之前的文章里说明了。所以我不多赘述了，如果你觉得这很麻烦，那么我有已经制作好的镜像和脚本，欢迎你的测试哦~

我这里给出一个群号，目前里面没有多少人，我制作的镜像、脚本和固件等等都会放在里面，这也是一个技术交流群，伸手党也欢迎加入哦，很多已经实现了的技术都可以拿来使用。
后期这个群可能会转为付费加群模式，新人加入的群费我会以红包的方式返回在群里大家抢，这样也算是一个良性循环，也杜绝了那些小号的存在哦。
其实LXD容器的N1实现我也在研究，但是一直遇到一个内核错误，我在尝试着解决它，欢迎各位大佬们加入我们以前讨论哈，这个LXD容器的N1实现之后我也会在群里放出。
群号：702993923
![qq群](/images/post/qq-code.jpg "qq群")

