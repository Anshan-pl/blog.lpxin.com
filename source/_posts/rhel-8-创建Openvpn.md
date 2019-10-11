---
title: rhel8搭建Openvpn并设置私网互通使用TCP代理
date: 2019-08-01 16:12:16
tags: [Redhat,Linux,shell,rhel]
categories: Redhat
---

<center>
接着上一篇的Redhat尝试<br/>
这次在rhel8上搭建一个Openvpn环境<br/>
使用的是Openvpn而不是openvpn-as
</center>
<!--more-->

# 添加epel扩展库
许多软件源都在扩展库里，所以我们先安装扩展库，注意：这里安装的rhel扩展库是rhel 7的而不是8，因为rhel8还在测试中，没有正式的EPEL存储库版本。
```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
dnf repolist epel

repo id                              repo name                                                                    status
*epel                                Extra Packages for Enterprise Linux 7 - x86_64                               13237
```
# Openvpn一键脚本
安装git，作用不用多说。
```
yum install git -y
cd ~
git clone https://github.com/Nyr/openvpn-install.git
cd openvpn-install
bash openvpn-install.sh


Welcome to this OpenVPN "road warrior" installer!
 I need to ask you a few questions before starting the setup.
 You can leave the default options and just press enter if you are ok with them.
 First, provide the IPv4 address of the network interface you want OpenVPN
 listening to.
 IP address: *192.168.199.152*   #VPN使用的IP地址
 This server is behind NAT. What is the public IPv4 address or hostname?
 Public IP address / hostname: *192.168.199.152*   #为了避免不必要的麻烦（修改主机名文件），这里请不要填域名，除非VPN在NAT内部，可以填写server的主机名。
 Which protocol do you want for OpenVPN connections?
    1) UDP (recommended)
    2) TCP
 Protocol [1-2]: *1*   #VPN使用的协议
 What port do you want OpenVPN listening to?
 Port: *1194*
 Which DNS do you want to use with the VPN?
    1) Current system resolvers
    2) 1.1.1.1
    3) Google
    4) OpenDNS
    5) Verisign
 DNS [1-5]: *1*   #与VPN一起使用的DNS解析服务器
 Finally, tell me your name for the client certificate.
 Please, use one word only, no special characters.
 Client name: *myvpn*   #VPN客户端名称
 Okay, that was all I needed. We are ready to set up your OpenVPN server now.
 Press any key to continue…
 Updating Subscription Management repositories.
 Updating Subscription Management repositories.
 Extra Packages for Enterprise Linux 7 - x86_64                                                                         189 kB/s |  16 MB     01:24    
 Last metadata expiration check: 0:00:54 ago on Wed 20 Mar 2019 07:23:31 PM EAT.
 Package epel-release-7-11.noarch is already installed.
 Dependencies resolved.
 Nothing to do.
 Complete!
 Updating Subscription Management repositories.
 Updating Subscription Management repositories.
 Waiting for process with pid 1906 to finish.
 Package iptables-1.8.0-11.el8.x86_64 is already installed.
 Package openssl-1:1.1.1-6.el8.x86_64 is already installed.
 Package ca-certificates-2018.2.24-6.el8.noarch is already installed.
 Dependencies resolved.
  Package                           Arch                    Version                           Repository                                           Size
 Installing:
  openvpn                           x86_64                  2.4.7-1.el7                       epel                                                522 k
 Installing dependencies:
  pkcs11-helper                     x86_64                  1.11-3.el7                        epel                                                 56 k
  libnsl                            x86_64                  2.28-18.el8                       rhel-8-for-x86_64-baseos-beta-rpms                   84 k
  compat-openssl10                  x86_64                  1:1.0.2o-3.el8                    rhel-8-for-x86_64-baseos-beta-rpms                  1.1 M
 Transaction Summary
 Install  4 Packages
 Total download size: 1.8 M
 Installed size: 4.6 M
 Downloading Packages:
 (1/4): pkcs11-helper-1.11-3.el7.x86_64.rpm                                                                              34 kB/s |  56 kB     00:01    
 (2/4): openvpn-2.4.7-1.el7.x86_64.rpm                                                                                  191 kB/s | 522 kB     00:02    
 (3/4): libnsl-2.28-18.el8.x86_64.rpm                                                                                    26 kB/s |  84 kB     00:03    
 (4/4): compat-openssl10-1.0.2o-3.el8.x86_64.rpm
.......................  

```

如果一切顺利，那么你应该得到下面的回显：
```
Check that the request matches the signature
 Signature ok
 The Subject's Distinguished Name is as follows
 commonName            :ASN.1 12:'computingforgeeks'
 Certificate is to be certified until Mar 17 16:24:47 2029 GMT (3650 days)
 Write out database with 1 new entries
 Data Base Updated
 Using SSL: openssl OpenSSL 1.1.1 FIPS  11 Sep 2018
 Using configuration from /etc/openvpn/easy-rsa/pki/safessl-easyrsa.cnf
 Can't load /etc/openvpn/easy-rsa/pki/.rnd into RNG
 140135296710464:error:2406F079:random number generator:RAND_load_file:Cannot open file:crypto/rand/randfile.c:90:Filename=/etc/openvpn/easy-rsa/pki/.rnd
 An updated CRL has been created.
 CRL file: /etc/openvpn/easy-rsa/pki/crl.pem
 788
 success
 success
 success
 success
 success
 success
 612
 Created symlink /etc/systemd/system/multi-user.target.wants/openvpn@server.service → /usr/lib/systemd/system/openvpn@.service.
 Finished!
 Your client configuration is available at: /root/computingforgeeks.ovpn
 If you want to add more clients, you simply need to run this script again!
```

# 客户端连接
## Centos7
假设你拥有一个Centos7，并且安装了epel扩展库，否则先安装epel
`yum install epel-release -y`

### 安装Openvpn客户端

```
yum install openvpn.x86_64 -y
```
### 连接Openvpn服务
使用FTP或者SCP服务将刚刚生成的配置文件导入到Centos的root目录下
我这里使用SCP命令导入
```
scp root@192.168.199.152:/root/myvpn.ovpn myvpn.ovpn
```
导入成功后，使用下面的命令连接
```
openvpn --config /root/myvpn.ovpn

Fri Aug  2 00:06:43 2019 Unrecognized option or missing or extra parameter(s) in /root/anshan_client.ovpn:14: block-outside-dns (2.4.7)
Fri Aug  2 00:06:43 2019 OpenVPN 2.4.7 x86_64-redhat-linux-gnu [Fedora EPEL patched] [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Feb 20 2019
Fri Aug  2 00:06:43 2019 library versions: OpenSSL 1.0.2k-fips  26 Jan 2017, LZO 2.06
Fri Aug  2 00:06:43 2019 Outgoing Control Channel Authentication: Using 512 bit message hash 'SHA512' for HMAC authentication
Fri Aug  2 00:06:43 2019 Incoming Control Channel Authentication: Using 512 bit message hash 'SHA512' for HMAC authentication
Fri Aug  2 00:06:43 2019 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:06:43 2019 Socket Buffers: R=[212992->212992] S=[212992->212992]
Fri Aug  2 00:06:43 2019 UDP link local: (not bound)
Fri Aug  2 00:06:43 2019 UDP link remote: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:06:43 2019 TLS: Initial packet from [AF_INET]192.168.199.152:1194, sid=9d04937e 075c4911
Fri Aug  2 00:06:43 2019 VERIFY OK: depth=1, CN=ChangeMe
Fri Aug  2 00:06:43 2019 VERIFY KU OK
Fri Aug  2 00:06:43 2019 Validating certificate extended key usage
Fri Aug  2 00:06:43 2019 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Fri Aug  2 00:06:43 2019 VERIFY EKU OK
Fri Aug  2 00:06:43 2019 VERIFY OK: depth=0, CN=server
Fri Aug  2 00:06:43 2019 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Fri Aug  2 00:06:43 2019 [server] Peer Connection Initiated with [AF_INET]192.168.199.152:1194
Fri Aug  2 00:06:44 2019 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Fri Aug  2 00:06:44 2019 PUSH: Received control message: 'PUSH_REPLY,redirect-gateway def1 bypass-dhcp,dhcp-option DNS 192.168.199.1,route-gateway 10.8.0.1,topology subnet,ping 10,ping-restart 120,ifconfig 10.8.0.2 255.255.255.0,peer-id 1,cipher AES-256-GCM'
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: timers and/or timeouts modified
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: --ifconfig/up options modified
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: route options modified
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: route-related options modified
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: peer-id set
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: adjusting link_mtu to 1624
Fri Aug  2 00:06:44 2019 OPTIONS IMPORT: data channel crypto options modified
Fri Aug  2 00:06:44 2019 Data Channel: using negotiated cipher 'AES-256-GCM'
Fri Aug  2 00:06:44 2019 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:06:44 2019 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:06:44 2019 ROUTE_GATEWAY 192.168.199.1/255.255.255.0 IFACE=ens33 HWADDR=00:0c:29:8e:79:87
Fri Aug  2 00:06:44 2019 TUN/TAP device tun0 opened
Fri Aug  2 00:06:44 2019 TUN/TAP TX queue length set to 100
Fri Aug  2 00:06:44 2019 /sbin/ip link set dev tun0 up mtu 1500
Fri Aug  2 00:06:44 2019 /sbin/ip addr add dev tun0 10.8.0.2/24 broadcast 10.8.0.255
Fri Aug  2 00:06:44 2019 /sbin/ip route add 192.168.199.152/32 dev ens33
Fri Aug  2 00:06:44 2019 /sbin/ip route add 0.0.0.0/1 via 10.8.0.1
Fri Aug  2 00:06:44 2019 /sbin/ip route add 128.0.0.0/1 via 10.8.0.1
Fri Aug  2 00:06:44 2019 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Fri Aug  2 00:06:44 2019 Initialization Sequence Completed
Fri Aug  2 00:10:29 2019 [server] Inactivity timeout (--ping-restart), restarting
Fri Aug  2 00:10:29 2019 SIGUSR1[soft,ping-restart] received, process restarting
Fri Aug  2 00:10:29 2019 Restart pause, 5 second(s)
Fri Aug  2 00:10:34 2019 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:10:34 2019 Socket Buffers: R=[212992->212992] S=[212992->212992]
Fri Aug  2 00:10:34 2019 UDP link local: (not bound)
Fri Aug  2 00:10:34 2019 UDP link remote: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:10:34 2019 TLS: Initial packet from [AF_INET]192.168.199.152:1194, sid=a1a17928 d56338df
Fri Aug  2 00:10:34 2019 VERIFY OK: depth=1, CN=ChangeMe
Fri Aug  2 00:10:34 2019 VERIFY KU OK
Fri Aug  2 00:10:34 2019 Validating certificate extended key usage
Fri Aug  2 00:10:34 2019 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Fri Aug  2 00:10:34 2019 VERIFY EKU OK
Fri Aug  2 00:10:34 2019 VERIFY OK: depth=0, CN=server
Fri Aug  2 00:10:34 2019 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Fri Aug  2 00:10:34 2019 [server] Peer Connection Initiated with [AF_INET]192.168.199.152:1194
Fri Aug  2 00:10:35 2019 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Fri Aug  2 00:10:35 2019 PUSH: Received control message: 'PUSH_REPLY,redirect-gateway def1 bypass-dhcp,dhcp-option DNS 192.168.199.1,route-gateway 10.8.0.1,topology subnet,ping 10,ping-restart 120,ifconfig 10.8.0.2 255.255.255.0,peer-id 1,cipher AES-256-GCM'
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: timers and/or timeouts modified
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: --ifconfig/up options modified
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: route options modified
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: route-related options modified
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: peer-id set
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: adjusting link_mtu to 1624
Fri Aug  2 00:10:35 2019 OPTIONS IMPORT: data channel crypto options modified
Fri Aug  2 00:10:35 2019 Data Channel: using negotiated cipher 'AES-256-GCM'
Fri Aug  2 00:10:35 2019 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:10:35 2019 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:10:35 2019 Preserving previous TUN/TAP instance: tun0
Fri Aug  2 00:10:35 2019 Initialization Sequence Completed
Fri Aug  2 00:14:37 2019 [server] Inactivity timeout (--ping-restart), restarting
Fri Aug  2 00:14:37 2019 SIGUSR1[soft,ping-restart] received, process restarting
Fri Aug  2 00:14:37 2019 Restart pause, 5 second(s)
Fri Aug  2 00:14:42 2019 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:14:42 2019 Socket Buffers: R=[212992->212992] S=[212992->212992]
Fri Aug  2 00:14:42 2019 UDP link local: (not bound)
Fri Aug  2 00:14:42 2019 UDP link remote: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:14:42 2019 TLS: Initial packet from [AF_INET]192.168.199.152:1194, sid=4fca5dd0 9fc10ee4
Fri Aug  2 00:14:42 2019 VERIFY OK: depth=1, CN=ChangeMe
Fri Aug  2 00:14:42 2019 VERIFY KU OK
Fri Aug  2 00:14:42 2019 Validating certificate extended key usage
Fri Aug  2 00:14:42 2019 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Fri Aug  2 00:14:42 2019 VERIFY EKU OK
Fri Aug  2 00:14:42 2019 VERIFY OK: depth=0, CN=server
Fri Aug  2 00:14:42 2019 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Fri Aug  2 00:14:42 2019 [server] Peer Connection Initiated with [AF_INET]192.168.199.152:1194
Fri Aug  2 00:14:43 2019 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Fri Aug  2 00:14:43 2019 PUSH: Received control message: 'PUSH_REPLY,redirect-gateway def1 bypass-dhcp,dhcp-option DNS 192.168.199.1,route-gateway 10.8.0.1,topology subnet,ping 10,ping-restart 120,ifconfig 10.8.0.2 255.255.255.0,peer-id 1,cipher AES-256-GCM'
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: timers and/or timeouts modified
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: --ifconfig/up options modified
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: route options modified
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: route-related options modified
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: peer-id set
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: adjusting link_mtu to 1624
Fri Aug  2 00:14:43 2019 OPTIONS IMPORT: data channel crypto options modified
Fri Aug  2 00:14:43 2019 Data Channel: using negotiated cipher 'AES-256-GCM'
Fri Aug  2 00:14:43 2019 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:14:43 2019 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:14:43 2019 Preserving previous TUN/TAP instance: tun0
Fri Aug  2 00:14:43 2019 Initialization Sequence Completed
Fri Aug  2 00:18:46 2019 [server] Inactivity timeout (--ping-restart), restarting
Fri Aug  2 00:18:46 2019 SIGUSR1[soft,ping-restart] received, process restarting
Fri Aug  2 00:18:46 2019 Restart pause, 5 second(s)
Fri Aug  2 00:18:51 2019 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:18:51 2019 Socket Buffers: R=[212992->212992] S=[212992->212992]
Fri Aug  2 00:18:51 2019 UDP link local: (not bound)
Fri Aug  2 00:18:51 2019 UDP link remote: [AF_INET]192.168.199.152:1194
Fri Aug  2 00:18:51 2019 TLS: Initial packet from [AF_INET]192.168.199.152:1194, sid=49312dc4 4bc07132
Fri Aug  2 00:18:51 2019 VERIFY OK: depth=1, CN=ChangeMe
Fri Aug  2 00:18:51 2019 VERIFY KU OK
Fri Aug  2 00:18:51 2019 Validating certificate extended key usage
Fri Aug  2 00:18:51 2019 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
Fri Aug  2 00:18:51 2019 VERIFY EKU OK
Fri Aug  2 00:18:51 2019 VERIFY OK: depth=0, CN=server
Fri Aug  2 00:18:51 2019 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Fri Aug  2 00:18:51 2019 [server] Peer Connection Initiated with [AF_INET]192.168.199.152:1194
Fri Aug  2 00:18:52 2019 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Fri Aug  2 00:18:52 2019 PUSH: Received control message: 'PUSH_REPLY,redirect-gateway def1 bypass-dhcp,dhcp-option DNS 192.168.199.1,route-gateway 10.8.0.1,topology subnet,ping 10,ping-restart 120,ifconfig 10.8.0.2 255.255.255.0,peer-id 1,cipher AES-256-GCM'
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: timers and/or timeouts modified
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: --ifconfig/up options modified
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: route options modified
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: route-related options modified
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: --ip-win32 and/or --dhcp-option options modified
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: peer-id set
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: adjusting link_mtu to 1624
Fri Aug  2 00:18:52 2019 OPTIONS IMPORT: data channel crypto options modified
Fri Aug  2 00:18:52 2019 Data Channel: using negotiated cipher 'AES-256-GCM'
Fri Aug  2 00:18:52 2019 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:18:52 2019 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Fri Aug  2 00:18:52 2019 Preserving previous TUN/TAP instance: tun0
Fri Aug  2 00:18:52 2019 Initialization Sequence Completed
```
Openvpn在服务器上会创建一个虚拟接口tun0，由Openvpn客户端的子网使用，此接口的默认子网是：10.8.0.0/24，OpenVPN服务器将被分配10.8.0.1这个IP地址

如果没有问题， 你可能会得到和我类似的日志输出，但是这个命令会一直占用此shell等待输入，所以使用下面的命令来将输出重定向。
```
openvpn --config /root/anshan_client.ovpn > vpn.log 2>&1 &
```
当然，你还可以使用nohup或者screen来达到这个效果，这取决于你。
使用ifconfig和ping命令来检查连接是否正常
```
ifconfig

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.199.220  netmask 255.255.255.0  broadcast 192.168.199.255
        inet6 fe80::f13d:60e2:67f4:6078  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:8e:79:87  txqueuelen 1000  (Ethernet)
        RX packets 112497  bytes 149884835 (142.9 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 35516  bytes 9351951 (8.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.8.0.2  netmask 255.255.255.0  destination 10.8.0.2
        inet6 fe80::7049:55ee:8978:2562  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 100  (UNSPEC)
        RX packets 2  bytes 168 (168.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5  bytes 312 (312.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


ping -c 5 -I 10.8.0.2 baidu.com

PING baidu.com (39.156.69.79) from 10.8.0.2 : 56(84) bytes of data.
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=1 ttl=49 time=29.2 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=2 ttl=49 time=28.2 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=3 ttl=49 time=28.1 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=4 ttl=49 time=28.7 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=5 ttl=49 time=29.0 ms

--- baidu.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4007ms
rtt min/avg/max/mdev = 28.136/28.700/29.264/0.451 ms
```
Centos7连接成功，注意，Openvpn默认一个用户只能有一个会话。

# 局域网设备访问外网
假设你的Openvpn客户端是没有外网访问权限，希望通过VPN来访问外网，那么你还需要添加一条静态路由
```
route add default gw 10.8.0.1
route -n #查看路由表 
```
这里我假设你的VPN子网段是10.8.0.0/24，否则请根据自己的情况修改，Openvpn默认dhcp网段是为10.8.0.0
使用ping命令检测是否连通外网
```
ping 119.29.29.29 -c 5

PING 119.29.29.29 (119.29.29.29) 56(84) bytes of data.
64 bytes from 119.29.29.29: icmp_seq=1 ttl=49 time=28.0 ms
64 bytes from 119.29.29.29: icmp_seq=2 ttl=49 time=28.6 ms
64 bytes from 119.29.29.29: icmp_seq=3 ttl=49 time=27.9 ms
64 bytes from 119.29.29.29: icmp_seq=4 ttl=49 time=26.9 ms
64 bytes from 119.29.29.29: icmp_seq=5 ttl=49 time=28.5 ms

--- 119.29.29.29 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 26.936/28.024/28.626/0.642 ms
```
这里只能ping一个外网的IP地址，你直接ping域名是不通的，因为你的DNS服务配置还没有修改。
```
echo "nameserver 119.29.29.29" >> /etc/resolv.conf
```
DNS服务IP可以不相同， 建议直接在文件内把原来的DNS删除，重新添加，否则域名解析时间会较长。

# 客户端访问服务端后的私网设备
这里我用的是最傻瓜的方式，配置静态路由。
在客户端上配置一条到达私网段且网关为VPN服务器的路由，
在需要访问到的设备上配置一条到VPN子网段且网关为VPN服务器的路由。

不会画网络拓扑图。。。只能文字描述下网络环境：
Openvpn服务器：
```
eth0 192.168.199.10/24
eth1 192.168.106.130/24
tun0 10.8.0.1/24
tun0为VPN创建的虚拟设备
```

私网设备：
```
eth0: 192.168.106.131/24
```

Openvpn客户机：
```
eth0: 192.168.199.11/24
tun0: 10.8.0.2/24
```
在连接上VPN的客户机配置如下
```
route add -net 192.168.106.0 netmask 255.255.255.0 gw 10.8.0.1
```

在私网设备上配置如下：
```
route add -net 192.168.106.0 netmask 255.255.255.0 gw 192.168.106.130
```

单单配置好路由也是不够的，在VPN服务端上还有一层防火墙阻挡了通信，方便起见直接关闭防火墙:
`systemctl stop firewalld`

在客户机或者私网设备上，现在两个网段都可以相互ping通了，原理只是在两个网段上添加路由，网关指向为可以同时连通两个网段的设备即可。

# TCP代理的实现
使用TCP代理之后，可以让同网段的所有设备通过代理服务器访问其所有的网段。
直接使用github上的开源项目goproxy
比如这里，我直接使用http一级代理
```
.\proxy http -t tcp -p "0.0.0.0:38080"
```
goproxy具体如何安装和使用，请直接阅览官方文档，在其官方的库中release上有编译好的各个系统对应的包，下载使用即可。
