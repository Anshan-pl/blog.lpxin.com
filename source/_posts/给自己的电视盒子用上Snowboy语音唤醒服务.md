---
title: 给自己的电视盒子用上Snowboy语音唤醒服务
date: 2019-06-16 09:30:51
tags: [Snowboy,N1,armbian,Linuxi,aarch64,armv8]
categories: Snowboy
---

<center>
利用自己手上的可以刷linux系统的设备来进行语音唤醒<br/>
可以实现语音控制家电，运行指定服务等等<br/>
百度的语音唤醒功能也是基于此服务哦<br/>
所以对应的唤醒词文件也是可以相互替换的<br/>
</center>
<!--more-->

## Snowboy
Snowboy是一个高度可定制的热门词检测引擎，它实时嵌入并始终与Raspberry Pi，（Ubuntu）Linux和Mac OS X兼容（即使在离线时）。
这是官方对于Snowboy的描述，比较抽象。我简单解释下：
它可以自定义唤醒词，如iphone的“Hey Siri”，小米的“小爱同学”，甚至是钢铁侠中的“Jarvis”，都可以作为你的自定义唤醒词。
它的机制是让服务不断的收集来着音频输入设备的声音，并判断是否是唤醒词，每0.03秒执行一次，不断监听作为信号触发的动作。

最为幸运的一点是，Snowboy的运行不依赖于网络，这意味着他极大的保护了你的隐私安全！
并且它具有轻量，嵌入式和兼容性高的优势！

## 个人唤醒词文件获取
Snowboy将唤醒词分为两大类，.pmdl和.umdl，其中
.pmdl是个人模型
.umdl是通用模型
简单来说，.pmdl是个人免费版本，它只对你的声音做出响应，然后根据我的实践，它会对任何声音做出响应，只是灵敏度问题。。。
而.umdl是付费版本，会对所有人呼叫唤醒词做出相应，我没有尝试过。

而获取个人版本非常简单
[进入Snoboy官网](https://snowboy.kitt.ai/dashboard)
登陆后点击Create Hotword，也就是创建你自己的唤醒词，Sonwboy把唤醒词称之为热门词。
输入你的唤醒词，你的年龄范围，最后的Personal Comment不用填，点击Record my voice。
点击Record后对着麦克风说出三遍你的唤醒词，这里需要三次音频输入。
继续点击Test the model后你就得到你的个人唤醒词模型了，下载到本地，scp到你的设备上，继续进行Sonwboy的安装。
`scp 你的唤醒词文件 root@xxx.xx.xx.xx:/root/demo.pmdl`

## Snowboy安装
### 安装依赖
这里我使用的绑定是python，如果你想用java或者go等其他开发语音，那么依赖可能有所不同。
这里不得不提一点，我的麦克风采样率是48000，不向下兼容，而Snowboy默认的采样率是16000，且不支持修改，当然这是一个开源程序，如果自行有些编译也是可以的。官方给出了一个解决方法，在python绑定实例中有一个demo_arecord.py解决了这个问题，所以我这里直接使用python而不是python3.

`apt update && apt-get install python-pyaudio python-dev sox swig libatlas-base-dev -y`

我在使用N1进行安装的时候遇到了一个Read-only file system错误，如下
```
dpkg: error: unable to create new file '/var/lib/dpkg/status-new': Read-only file system
E: Sub-process /usr/bin/dpkg returned an error code (2)
W: Problem unlinking the file /var/cache/apt/pkgcache.bin - pkgDPkgPM::Go (30: Read-only file system)

```
这是armbian安装到emmc引发的错误，可能是版本或者是其他的问题，我没有仔细寻找，因为它可以通过重启解决。如果你也遇到了这个问题，那么请继续跟着我，
```
reboot
dpkg --configure -a
apt --fix-broken install -y
apt-get install python-pyaudio python-dev sox swig libatlas-base-dev -y

```

### 麦克风测试
既然是语音唤醒服务，那么有一个音频输入设备也就是麦克风是必要的，按照官方文档，接下来你需要测试麦克风
`rec temp.wav`
如果你和我一样使用的是N1或者其他不具有3.5mm音频输入口的设备，那么很不幸，你将获得如下错误：
```
rec WARN alsa: can't encode 0-bit Unknown or not applicable
rec FAIL formats: can't open input  `default': snd_pcm_open error: No such file or directory
```
因为这里的播放设备默认是hw0，也就是3.5mm音频输入接口，所以我们需要一个usb麦克风，这里为了避嫌，我不给出链接了，淘宝树莓派usb麦克风，你将得到你想要的，他很便宜，是否好用，由你而定。

假设你购买了这个usb麦克风，即使它应该是“即插即用”，但是在这你也不得不重启设备。
这里给出三个检测设备是否成功识别麦克风的方法：

#### lsusb
```
lsusb

Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 002: ID 0c76:160a JMTek, LLC.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

```
这里中间的输出有设备型号，那么就证明你的usb麦克风已经被检测到了，如果没有，那么你的麦克风可能不是linux免驱的。
这里我最后一行的输出是usb2.0，你的可能会是usb1.1，这是由于不同的音频芯片造成的，PCM2902芯片对应是usb1.1，而CM108对应的是usb2.0，这里你也无需太过纠结，因为对于aarch64设备来说，usb1.1的性能也完全足够了。如果你不太确定买哪一款，可以加文章最下方的Q群，我在群里推荐了一款我自用的，它的颜值够高，价格也非常便宜，并且是CM108音频芯片。

#### arecord -l
```
arecord -l

**** List of CAPTURE Hardware Devices ****
card 1: Device [USB Audio Device], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

#### aplay -l
```
aplay -l

**** List of PLAYBACK Hardware Devices ****
card 0: HDMI [HDMI], device 0: meson-aiu-i2s.4.auto-i2s-hifi i2s-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

#### dmesg | grep USB
```
dmesg | grep USB
[    2.987572] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    2.997752] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    3.120263] usbhid: USB HID core driver
[    3.301982] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 1
[    3.325007] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.01
[    3.332935] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    3.356556] hub 1-0:1.0: USB hub found
[    3.369248] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 2
[    3.376836] xhci-hcd xhci-hcd.0.auto: Host supports USB 3.0  SuperSpeed
[    3.391486] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 5.01
[    3.399600] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    3.423177] hub 2-0:1.0: USB hub found
[  993.678086] usb 1-2: new full-speed USB device number 2 using xhci-hcd
[  993.826875] usb 1-2: New USB device found, idVendor=0c76, idProduct=160a, bcdDevice= 1.00
[  993.826894] usb 1-2: New USB device strings: Mfr=0, Product=1, SerialNumber=0
[  993.826905] usb 1-2: Product: USB Audio Device
[  993.842993] input: USB Audio Device as /devices/platform/soc/soc:usb@c9000000/c9000000.dwc3/xhci-hcd.0.auto/usb1/1-2/1-2:1.2/0003:0C76:160A.0001/input/input4
[  993.902753] hid-generic 0003:0C76:160A.0001: input,hidraw0: USB HID v1.00 Device [USB Audio Device] on usb-xhci-hcd.0.auto-2/input2
```
看到 input就证明是正常的

### 修改默认音频设备
如果你在上一步麦克风检测时一切正常，那么请跳过这一步，它是对于和我一样使用usb麦克风的人。
```
vim ~/.asoundrc
pcm.!default {
  type asym
   playback.pcm {
     type plug
     slave.pcm "hw:0,0"
   }
   capture.pcm {
     type plug
     slave.pcm "hw:1,0"  #请注意，这里需要修改为你的设备号，如果没有意外的话将和我一致。
   }
}
```

接下来，运行
rec temp.wav
你会神奇的发现，获得了一个跳跃的字符行。

### 克隆Snowboy源码
`git clone https://github.com/Kitt-AI/snowboy.git`

### 编译Python绑定
这里不选择Python3的原因上面已经提到了，如果在这里你使用了Python3并且你的麦克风不支持16000采样率，那么在最后一步你将得到一个IOError: [Errno -9997] Invalid sample rate错误。

```
cd snowboy/swig/Python
make
```
你可能会得到如下报错，但是请不要担心，因为，我已经将所有的坑踩了一遍，如果你是按照我的教程来进行的，那么你不会遇到这个报错，因为这是告诉你缺少python的开发模块，请安装python-dev。
```
snowboy-detect-swig.cc:176:11: fatal error: Python.h: No such file or directory
 # include <Python.h>
           ^~~~~~~~~~
compilation terminated.
Makefile:70: recipe for target 'snowboy-detect-swig.o' failed
make: *** [snowboy-detect-swig.o] Error 1

```

接着上次的编译进程，继续编译，你会得到如下报错：

```
/usr/bin/ld: ../..//lib/ubuntu64/libsnowboy-detect.a(snowboy-detect.o): Relocations in generic ELF (EM: 62)
/usr/bin/ld: ../..//lib/ubuntu64/libsnowboy-detect.a(snowboy-detect.o): Relocations in generic ELF (EM: 62)
/usr/bin/ld: ../..//lib/ubuntu64/libsnowboy-detect.a(snowboy-detect.o): Relocations in generic ELF (EM: 62)
/usr/bin/ld: ../..//lib/ubuntu64/libsnowboy-detect.a(snowboy-detect.o): Relocations in generic ELF (EM: 62)
../..//lib/ubuntu64/libsnowboy-detect.a: error adding symbols: File in wrong format
collect2: error: ld returned 1 exit status
Makefile:73: recipe for target '_snowboydetect.so' failed
make: *** [_snowboydetect.so] Error 1
```
这是由于makefile里并没有aarch64，或许是snowboy官方不支持aarch64，也可能是官方的一个小bug，但是没关系，请继续跟着我的脚步：

```
vim Makefile

else
  CXX := g++
  PYINC := $(shell python3-config --cflags)
  PYLIBS := $(shell python3-config --ldflags)
  SWIGFLAGS := -shared
  CXXFLAGS += -std=c++0x
  # Make sure you have Atlas installed. You can statically link Atlas if you
  # would like to be able to move the library to a machine without Atlas.
  ifneq ("$(ldconfig -p | grep lapack_atlas)","")
    LDLIBS := -lm -ldl -lf77blas -lcblas -llapack_atlas -latlas
  else
    LDLIBS := -lm -ldl -lf77blas -lcblas -llapack -latlas
  endif
  SNOWBOYDETECTLIBFILE = $(TOPDIR)/lib/ubuntu64/libsnowboy-detect.a
  ifneq (,$(findstring arm,$(shell uname -m)))
    SNOWBOYDETECTLIBFILE = $(TOPDIR)/lib/rpi/libsnowboy-detect.a
    ifeq ($(findstring fc,$(shell uname -r)), fc)
      SNOWBOYDETECTLIBFILE = $(TOPDIR)/lib/fedora25-armv7/libsnowboy-detect.a
      LDLIBS := -L/usr/lib/atlas -lm -ldl -lsatlas
    endif
  endif
  #请添加下面的代码来对aarch64进行支持
  ifneq (,$(findstring aarch64,$(shell uname -m)))
    SNOWBOYDETECTLIBFILE = $(TOPDIR)/lib/aarch64-ubuntu1604/libsnowboy-detect.a
  endif

endif

```
保存后，继续进行编译
`make`
如果没有意外，你会得到如下输出：
```
g++ -I../../ -O3 -fPIC -D_GLIBCXX_USE_CXX11_ABI=0 -std=c++0x  -shared snowboy-detect-swig.o \
../..//lib/aarch64-ubuntu1604/libsnowboy-detect.a -L/usr/lib/python3.6/config-3.6m-aarch64-linux-gnu -L/usr/lib -lpython3.6m -lpthread -ldl  -lutil -lm  -Xlinker -export-dynamic -Wl,-O1 -Wl,-Bsymbolic-functions -lm -ldl -lf77blas -lcblas -llapack -latlas -o _snowboydetect.so
```
接下来，回到examples/Python目录下， 有demo.py文件：
`cd ../../examples/Python`
如何使用，官方上有非常详细的介绍。
http://docs.kitt.ai/snowboy/#running-a-demo
我只做一个简单的演示
```
python demo.py ~/demo.pmdl

```
如果没有意外对着你的麦克风说出你自定义的唤醒词，你的屏幕上会打印出一个INPUT

如果这里你遇到了
IOError: [Errno -9997] Invalid sample rate
那么请使用demo_arecord.py。
`python demo_arecord.py ~/demo.pmdl`

## 灵敏度调整
对于Python实例，它应该在文件的第27行
```
# capture SIGINT signal, e.g., Ctrl+C
signal.signal(signal.SIGINT, signal_handler)

detector = snowboydecoder.HotwordDetector(model, sensitivity=0.5)
print('Listening... Press Ctrl+C to exit')
```
这里将sensitivity=0.5改成你想要的灵敏度即可，默认是0.5，范围在1以内。
如果你是多模型回调的，官方demo的灵敏度是模型数量*0.5的，但是根据我的测试，灵敏度不需要那么高，单个灵敏度是多少，这里就填多少。

## 控制家庭电路
这里说是控制家庭电路有些夸张了，因为我使用的方法是一个LCUS-1型USB继电器模块来实现电路控制的，而这款继电器的电压范围是0-36V，家庭电路的电压最高可以240V，所以这里能控制的只有小电压的电路了， 比如led灯之类的。
假设这里你使用的是和我一样的LCUS-1型继电器，那么可以参考我的控制代码
```
import snowboydecoder_arecord
import sys
import signal

interrupted = False


def signal_handler(signal, frame):
    global interrupted
    interrupted = True


def interrupt_callback():
    global interrupted
    return interrupted

if len(sys.argv) == 1:
    print("Error: need to specify model name")
    print("Usage: python demo.py your.model")
    sys.exit(-1)

model = sys.argv[1]

# capture SIGINT signal, e.g., Ctrl+C
signal.signal(signal.SIGINT, signal_handler)

detector = snowboydecoder_arecord.HotwordDetector(model, sensitivity=0.5)
print('Listening... Press Ctrl+C to exit')

def open_lamp():
    with open("/root/lamp_bool", 'r') as fr:
        lamp_bool = fr.read()
    if lamp_bool != "True":
        with open("/dev/ttyUSB0", 'w') as fw:
            fw.write("\xA0\x01\x01\xA2")
        with open("/root/lamp_bool", 'w') as fw:
            fw.write("True")
    else:
        with open("/dev/ttyUSB0", 'w') as fw:
            fw.write("\xA0\x01\x00\xA1")
        with open("/root/lamp_bool", 'w') as fw:
            fw.write("False")
# main loop
detector.start(detected_callback=open_lamp,
#detector.start(detected_callback=snowboydecoder_arecord.play_audio_file,
               interrupt_check=interrupt_callback,
               sleep_time=0.03)

detector.terminate()

```
顺便放上一张，自制的星图效果图，材料有限，时间仓促，可以说是惨不忍睹，漏光严重。。这个星图的电路控制就是使用的上述代码段。
想了解制作流程，或者控制电路更详细的方式，可以留言或私信.
![星图](/images/post/xintu.jpg "星图")

## 疑问解决
如果你在安装或者使用时，遇到了问题，可以添加下面的Q群，有很多群友或许已经遇到过了，我也会为你解答，
在评论区留音，如果是我遇到的问题，我会第一时间给你回复。
QQ群:702993923

![qqcode](/images/post/qq-code.jpg "qqcode")

**end**
