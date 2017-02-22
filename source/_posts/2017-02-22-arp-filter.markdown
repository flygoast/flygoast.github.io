---
layout: post
title: "ARP响应实例研究"
date: 2017-02-22 19:55:04 +0800
comments: true
categories: OpenStack
---
之前文章[<<ARP代理实例研究>>](http://www.just4coding.com/blog/2017/02/19/arp-proxy/)提到: 在默认配置下，只要ARP请求中的目标IP配置在本机，无论其是否配置在收到ARP请求数据包的接口上，Linux收包接口都会以身MAC地址发送ARP响应。若是不希望接口响应所有本机IP，可以通过修改`arp_ignore`参数来调整。

那么如果机器上多个网卡连接在同一交换机，这样多个网卡会收到相同的ARP请求。若多个网卡都回复，则客户端应该收到多个ARP响应。这种情况下，Linux是如何处理的呢？

<!--more-->

我们来实验来看一下。实验环境为VirtualBox HostOnly网络`vboxnet0`, 宿主机网关IP为:`192.168.33.1`, 虚拟机上有两个网卡接入该网络，IP分别为`192.168.33.10`和`192.168.33.11`。

首先在宿主机上清空`vboxnet0`的ARP缓存:
`arp -d -ivboxnet0 -a`
然后启动`tcpdump`抓包，在另一终端来访问`192.168.33.11`。
```plain
bogon:~ fenggu$ ping -c 1 192.168.33.11
PING 192.168.33.11 (192.168.33.11): 56 data bytes
64 bytes from 192.168.33.11: icmp_seq=0 ttl=64 time=0.563 ms

--- 192.168.33.11 ping statistics ---
1 packets transmitted, 1 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.563/0.563/0.563/0.000 ms
```
抓包结果:
```plain
sh-3.2# tcpdump -ivboxnet0 -nn arp or icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vboxnet0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:21:20.981561 ARP, Request who-has 192.168.33.11 tell 192.168.33.1, length 28
15:21:20.981890 ARP, Reply 192.168.33.11 is-at 08:00:27:e6:6a:de, length 46
15:21:20.981922 IP 192.168.33.1 > 192.168.33.11: ICMP echo request, id 42280, seq 0, length 64
15:21:20.982032 IP 192.168.33.11 > 192.168.33.1: ICMP echo reply, id 42280, seq 0, length 64
^C
4 packets captured
10 packets received by filter
0 packets dropped by kernel
```
抓包显示只收到了一个ARP响应。

虚拟机上查看网络地址:
```plain
[root@centos1 vagrant]# ip a
...
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:e6:6a:de brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.10/24 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fee6:6ade/64 scope link
       valid_lft forever preferred_lft forever
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:3c:1e:5b brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.11/24 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe3c:1e5b/64 scope link
       valid_lft forever preferred_lft forever
```
响应的MAC地址`08:00:27:e6:6a:de`为网卡`enp0s8`的地址。

哪个设备来响应ARP请求如何决定呢?此时，我们来看路由信息:
```plain
[root@centos1 vagrant]# ip route
default via 10.0.2.2 dev enp0s3  proto static  metric 100
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15  metric 100
192.168.33.0/24 dev enp0s8  proto kernel  scope link  src 192.168.33.10
192.168.33.0/24 dev enp0s9  proto kernel  scope link  src 192.168.33.11
```
此时`enp0s8`的路由条目优先级高。我们重启`enp0s8`设备，调整两条路由条目的顺序:
```plain
[root@centos1 vagrant]# ip link set down enp0s8
[root@centos1 vagrant]# ip link set up enp0s8
[root@centos1 vagrant]# ip route
default via 10.0.2.2 dev enp0s3  proto static  metric 100
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15  metric 100
192.168.33.0/24 dev enp0s9  proto kernel  scope link  src 192.168.33.11
192.168.33.0/24 dev enp0s8  proto kernel  scope link  src 192.168.33.10
```
此时在宿主机上，清空ARP缓存再次访问`192.168.33.11`，抓包结果为:
```plain
sh-3.2# tcpdump -ivboxnet0 -nn arp or icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vboxnet0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:30:57.113221 ARP, Request who-has 192.168.33.11 tell 192.168.33.1, length 28
15:30:57.113542 ARP, Reply 192.168.33.11 is-at 08:00:27:3c:1e:5b, length 46
15:30:57.113574 IP 192.168.33.1 > 192.168.33.11: ICMP echo request, id 4137, seq 0, length 64
15:30:57.113636 IP 192.168.33.11 > 192.168.33.1: ICMP echo reply, id 4137, seq 0, length 64
^C
4 packets captured
5 packets received by filter
0 packets dropped by kernel
```
响应的MAC地址`08:00:27:3c:1e:5b`为网卡`enp0s9`的地址。

在虚拟机上查看`rp_filter`选项:
```plain
[root@centos1 vagrant]# sysctl -a |grep rp_filter
net.ipv4.conf.all.arp_filter = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.arp_filter = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.enp0s3.arp_filter = 0
net.ipv4.conf.enp0s3.rp_filter = 1
net.ipv4.conf.enp0s8.arp_filter = 0
net.ipv4.conf.enp0s8.rp_filter = 1
net.ipv4.conf.enp0s9.arp_filter = 0
net.ipv4.conf.enp0s9.rp_filter = 1
net.ipv4.conf.lo.arp_filter = 0
net.ipv4.conf.lo.rp_filter = 0
```
可以发现设备的`rp_filter`选项是开启的。Linux在处理ARP数据包时会对源IP进行路由查找。在`rp_filter`开启的情况下，若路由找到的接口与接收ARP的接口不一致，该包就被丢弃了。我们第一次请求时，`enp0s8`的路由条目优先级更高，路由到的接口为`enp0s8`，`enp0s9`收到的ARP包被丢弃。我们调整完路由条目顺序后，`enp0s8`的ARP包就被丢弃了。

接下来我们在虚拟机上将`rp_filter`选项关闭:
```bash
sysctl -w net.ipv4.conf.enp0s8.rp_filter=0
sysctl -w net.ipv4.conf.enp0s9.rp_filter=0
sysctl -w net.ipv4.conf.all.rp_filter=0
```
再次在宿主机上清除缓存并访问`192.168.33.11`, 抓包结果:
```plain
sh-3.2# tcpdump -ivboxnet0 -nn arp or icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vboxnet0, link-type EN10MB (Ethernet), capture size 262144 bytes
16:05:28.756702 ARP, Request who-has 192.168.33.11 tell 192.168.33.1, length 28
16:05:28.756976 ARP, Reply 192.168.33.11 is-at 08:00:27:3c:1e:5b, length 46
16:05:28.757011 IP 192.168.33.1 > 192.168.33.11: ICMP echo request, id 34346, seq 0, length 64
16:05:28.757072 ARP, Reply 192.168.33.11 is-at 08:00:27:e6:6a:de, length 46
16:05:28.757163 IP 192.168.33.11 > 192.168.33.1: ICMP echo reply, id 34346, seq 0, length 64
^C
5 packets captured
5 packets received by filter
0 packets dropped by kernel
```
发现收到了两个ARP响应。

接着我们在虚拟机上开启`arp_filter`选项:
```plain
[root@centos1 vagrant]# sysctl -w net.ipv4.conf.enp0s8.arp_filter=1
net.ipv4.conf.enp0s8.arp_filter = 1
[root@centos1 vagrant]# sysctl -w net.ipv4.conf.enp0s9.arp_filter=1
net.ipv4.conf.enp0s9.arp_filter = 1
[root@centos1 vagrant]# sysctl -a |grep arp_filter
net.ipv4.conf.all.arp_filter = 0
net.ipv4.conf.default.arp_filter = 0
net.ipv4.conf.enp0s3.arp_filter = 0
net.ipv4.conf.enp0s8.arp_filter = 1
net.ipv4.conf.enp0s9.arp_filter = 1
net.ipv4.conf.lo.arp_filter = 0
```
再次从宿主机清缓存来访问，抓包结果:
```plain
sh-3.2# tcpdump -ivboxnet0 -nn arp or icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vboxnet0, link-type EN10MB (Ethernet), capture size 262144 bytes
16:09:34.975032 ARP, Request who-has 192.168.33.11 tell 192.168.33.1, length 28
16:09:34.975354 ARP, Reply 192.168.33.11 is-at 08:00:27:3c:1e:5b, length 46
16:09:34.975385 IP 192.168.33.1 > 192.168.33.11: ICMP echo request, id 44330, seq 0, length 64
16:09:34.975466 IP 192.168.33.11 > 192.168.33.1: ICMP echo reply, id 44330, seq 0, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```
响应的MAC地址`08:00:27:3c:1e:5b`为网卡`enp0s9`的地址。
再次将设备`enp0s9`重启:
```plain
[root@centos1 vagrant]# ip link set enp0s9 down
[root@centos1 vagrant]# ip link set enp0s9 up
[root@centos1 vagrant]# ip route
default via 10.0.2.2 dev enp0s3  proto static  metric 100
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15  metric 100
192.168.33.0/24 dev enp0s8  proto kernel  scope link  src 192.168.33.10
192.168.33.0/24 dev enp0s9  proto kernel  scope link  src 192.168.33.11
```
再次实验抓包结果如下:
```plain
sh-3.2# tcpdump -ivboxnet0 -nn arp or icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vboxnet0, link-type EN10MB (Ethernet), capture size 262144 bytes
16:12:30.150463 ARP, Request who-has 192.168.33.11 tell 192.168.33.1, length 28
16:12:30.150775 ARP, Reply 192.168.33.11 is-at 08:00:27:e6:6a:de, length 46
16:12:30.150800 IP 192.168.33.1 > 192.168.33.11: ICMP echo request, id 52010, seq 0, length 64
16:12:30.150988 IP 192.168.33.11 > 192.168.33.1: ICMP echo reply, id 52010, seq 0, length 64
^C
4 packets captured
5 packets received by filter
0 packets dropped by kernel
```
响应的MAC地址`08:00:27:3c:1e:5b`为网卡`enp0s8`的地址。与`rp_filter`参数的行为表现一致。

事实上，`arp_filter`选项开启时，也会对源IP进行路由查找。若找到的接口与接收ARP的接口不一致，包被丢弃。

`rp_filter`和`arp_filter`选项的逻辑相似，但控制的层次不一样。`rp_filter`控制整个路由系统的过滤机制，`arp_filter`控制ARP响应的过滤机制。

`rp_filter`和`arp_filter`的文档地址:
https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt

