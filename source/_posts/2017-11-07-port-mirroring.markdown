---
layout: post
title: "VMware vSphere平台端口镜像"
date: 2017-11-07 16:23:16 +0800
comments: true
categories: Virtualization
---
端口镜像是物理交换机上的一种特性，用于将交换机上某一端口的流量发送至其他目的地。镜像的流量通常可以用于网络调试、性能监测、安全防护等方面。

Cisco将端口镜像分为三种形式:

* SPAN(Switch Port Analyzer): 将流量从源端口镜像至同一交换机上的目的端口。
* RSPAN(Remote SPAN): 这种形式扩展了SPAN方式，允许镜像流量经过多个二层网络设备。RSPAN将流量镜像至特定VLAN，流经的交换机需要允许该VLAN通过，流量到达目标交换机时镜像至目的端口。
* ERSPAN(Encapsulated RSPAN): ERSPAN允许将流量镜像至三层网络设备。ERSPAN将原始流量封装到GRE(Generic Routing Encapsulation)中经由3层网络发送至目标设备。

VMware vSphere的VDS(Virtual Distributed Switch)也能够支持这些特性。VMware VDS有5种可用的端口镜像选项, 如图:

{% img /images/2017-11-07/1.png %}

<!--more-->

`分布式端口镜像`对应`SPAN`, 它将流量从一个端口镜像到同一分布式交换机的另一端口，两个端口还必须位于同一个宿主机上。

下面我们具体实验来说明，实验环境如图:

{% img /images/2017-11-07/2.png %}

虚拟机`t1`和`t2`位于同一个VDS:`DSwitch`上，我们将`t1`的流量镜像至`t2`。

我们在分布式交换机`DSwitch`上添加一个`分布式端口镜像`会话，将`状态`选为`已启用`。若目标端口不只单纯接收镜像的流量，还有自身的网络流量，则需要将`目标端口上的正常I/O`选为`允许`, 如图:

{% img /images/2017-11-07/3.png %}

接下来，选择`t1`的端口做为源端口, 如图:

{% img /images/2017-11-07/4.png %}

接着选择`t2`的端口做为目标端口, 如图:

{% img /images/2017-11-07/5.png %}


此时结果如图, 点击完成:

{% img /images/2017-11-07/6.png %}


我们在`t2`上使用tcpdump抓包, 在`t1`上执行PING操作。

在`t1`上执行PING操作:
```plain
[root@localhost ~]# ping 114.114.114.114 -c 2
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=67 time=27.7 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=63 time=26.9 ms

--- 114.114.114.114 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 26.924/27.341/27.759/0.449 ms
```
在`t2`上查看抓包结果:
```plain
[root@localhost ~]# tcpdump -iens160 -nn icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens160, link-type EN10MB (Ethernet), capture size 262144 bytes
01:44:33.311035 IP 10.95.48.150 > 114.114.114.114: ICMP echo request, id 18475, seq 1, length 64
01:44:33.338740 IP 114.114.114.114 > 10.95.48.150: ICMP echo reply, id 18475, seq 1, length 64
01:44:34.312885 IP 10.95.48.150 > 114.114.114.114: ICMP echo request, id 18475, seq 2, length 64
01:44:34.339760 IP 114.114.114.114 > 10.95.48.150: ICMP echo reply, id 18475, seq 2, length 64
```
可以看到`t2`上完整拿到了`t1`的流量。

`远程镜像源`和`远程镜像目的`两个选项对应`RSPAN`。当源端口和目的端口位于不同的分布式交换机或者ESXi宿主机上时，可以使用这两个选项。远程镜像源选项将源端口的流量以特定VLAN封装发送到指定的上行链路上，远程镜像目的选项再将特定VLAN的数据镜像到目的端口。这两个选项本身还涉及物理交换机配置VLAN，不再具体说明。

`已封装远程镜像(L3)源`对应`ERSPAN`, 可以将源端口流量通过三层网络发送到指定的目的设备。

下面，我们将`t1`的流量镜像到三层网络地址: `10.95.30.43`。

创建步骤与上边类似，配置完结果如图:

{% img /images/2017-11-07/7.png %}

我们在目标机器`10.95.30.43`上抓包，可以看到`t1`上的以太网帧数据以GRE形式封装镜像了过来(`10.95.30.31`为`t1`宿主机IP):
```plain
[root@localhost ~]# tcpdump -i br0 -e -nn proto GRE
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 65535 bytes
15:28:20.615095 2c:9d:1e:5c:27:98 > 9c:e3:74:d0:9f:d9, ethertype IPv4 (0x0800), length 102: 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, proto TEB (0x6558), length 68: 00:50:56:b1:2c:bb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.95.47.50 tell 10.95.47.49, length 46
15:28:20.640046 2c:9d:1e:5c:27:98 > 9c:e3:74:d0:9f:d9, ethertype IPv4 (0x0800), length 102: 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, proto TEB (0x6558), length 68: e8:61:1f:1b:9a:a8 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.95.31.252 tell 10.95.30.19, length 46
15:28:20.705853 2c:9d:1e:5c:27:98 > 9c:e3:74:d0:9f:d9, ethertype IPv4 (0x0800), length 102: 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, proto TEB (0x6558), length 68: 00:50:56:a0:34:ce > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.95.138.151 tell 10.95.46.20, length 46
15:28:20.727207 2c:9d:1e:5c:27:98 > 9c:e3:74:d0:9f:d9, ethertype IPv4 (0x0800), length 102: 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, proto TEB (0x6558), length 68: 6c:92:bf:58:5c:e0 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.95.31.252 tell 10.95.30.18, length 46
```
