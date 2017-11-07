---
layout: post
title: "VMware vSphere平台端口镜像"
date: 2017-11-07 16:23:16 +0800
comments: true
categories: Virtualization
---
端口镜像是物理交换机上的一种特性，用于将交换机上某一端口的流量发送至其他目的地。镜像的流量通常可以用于网络调试、性能监测、安全防护等方面。

Cisco将端口镜像分为三种形式:

* SPAN(Switch Port Analyzer): 将流量从源端口镜像至同一交换机上的目的端口
* RSPAN(Remote SPAN): 这种形式扩展了SPAN方式，允许将镜像流量经过多个二层网络设备。RSPAN将流量镜像至特定VLAN，流经的交换机需要允许该VLAN通过，流量到达目标交换机时镜像至目的端口。
* ERSPAN(Encapsulated RSPAN): ERSPAN允许将流量镜像至三层网络设备。ERSPAN将原始流量封装到GRE(Generic Routing Encapsulation)中经由3层网络发送至目标设备。

VMware vSphere的VDS(Virtual Distributed Switch)也能够支持这些特性。VMware VDS有5种可用的端口镜像选项, 如图:

{% img /images/2017-11-07/1.png %}

<!--more-->

分布式端口镜像对应SPAN, 它将流量从一个端口镜像到同一分布式交换机的另一端口，两个端口还必须位于同一个宿主机上。

下面我们具体实验来说明，实验环境如图:

{% img /images/2017-11-07/2.png %}

虚拟机`t1`和`t2`位于同一个VDS:`DSwitch`上，我们将`t1`的流量镜像至`t2`。

我们在分布式交换机DSwitch上依次完成如下步骤:

* 添加一个端口镜像会话
* 启用会话
* 选择t1的端口做为源端口
* 选择t2的端口做为目标端口
* 点击完成

此时结果如图:

{% img /images/2017-11-07/3.png %}


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

远程镜像源和远程镜像目的两个选项对应RSPAN。当源端口和目的端口位于不同的分布式交换机或者ESXi宿主机上时，可以使用这两个选项。远程镜像源选项将源端口的流量以特定VLAN封装发送到指定的上行链路上，远程镜像目的选项再将特定VLAN的数据镜像到目的端口。这两个选项本身还涉及物理交换机配置VLAN，不再具体说明。

已封装远程镜像(L3)源对应ERSPAN。可以将源端口流量通过三层网络发送到指定的目的设备。

下面，我们将`t1`的流量镜像到三层网络地址: `10.95.30.43`。

创建步骤与上边类似，配置完结果如图:

{% img /images/2017-11-07/4.png %}

我们在目标机器`10.95.30.43`上抓包，可以看到`t1`上的数据以GRE形式镜像了过来(`10.95.30.31`为`t1`宿主机IP):
```plain
[root@localhost ~]# tcpdump -i br0 -nn proto GRE && proto icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 65535 bytes
15:14:09.695807 IP 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, length 68: ARP, Request who-has 10.95.31.252 tell 10.95.30.18, length 46
15:14:09.747178 IP 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, length 68: ARP, Request who-has 10.95.31.252 tell 10.95.30.23, length 46
15:14:09.747384 IP 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, length 68: ARP, Request who-has 10.95.31.252 tell 10.95.30.16, length 46
15:14:09.768540 IP 10.95.30.31 > 10.95.30.43: GREv0, key=0x0, length 68: ARP, Request who-has 10.95.47.56 tell 10.95.47.57, length 46
```

