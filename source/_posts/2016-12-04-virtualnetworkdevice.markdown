---
layout: post
title: "虚拟网络设备"
date: 2016-12-04 18:21:36 +0800
comments: true
categories: OpenStack
---
OpenStack虚拟化网络实现中大量应用了多种虚拟网络设备，了解这些设备是理解OpenStack虚拟网络实现的基础，本文来简单介绍这些虚拟网络设备。

## TUN/TAP设备

TUN/TAP设备是Linux内核中实现的虚拟网卡。物理网卡是从物理线路上收发数据包，而TUN/TAP设备是从用户态应用程序上收发以太网帧或IP包。用户态进程对`/dev/net/tun`文件调用`open()`获取一个文件描述符，并调用`ioctl()`挂接到该设备上，接着通过读写该文件描述符从TUN/TAP设备的收发数据包。收发的数据包由用户态进程构造好。TUN和TAP设备的区别在于TUN设备收发的是IP包，而TAP设备收发的是以太网帧。

在进程中创建及使用TUN/TAP设备可以参考官方文档:
https://www.kernel.org/doc/Documentation/networking/tuntap.txt

可以使用iproute2工具包中的ip命令创建TUN/TAP设备, 如:
```bash
ip tuntap add dev tap0 mode tap
ip tuntap add dev tun0 mode tun
```

<!--more-->

通过`ip link`命令可以看到设备已经创建:
```bash
[root@compute1 ~]# ip link show tap0
23: tap0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 500
    link/ether a6:73:4e:90:f9:3e brd ff:ff:ff:ff:ff:ff
[root@compute1 ~]# ip link show tun0
24: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 500
    link/none
```

设备创建完成后可以和物理设备一样对TAP进行配置IP地址等操作，如:
```bash
[root@compute1 ~]# ip addr add 192.168.1.2/24 dev tap0
[root@compute1 ~]# ip link set dev tap0 up
[root@compute1 ~]# ip addr list dev tap0
23: tap0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN qlen 500
    link/ether 36:f2:68:6a:fd:6d brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.2/24 scope global tap0
       valid_lft forever preferred_lft forever
```
删除已创建的设备:
```bash
ip tuntap del dev tap0 mode tap
ip tuntap del dev tun0 mode tun
```

## Linux bridge

Linux bridge是虚拟二层交换设备, 它基于MAC地址对数据包在bridge的port间进行转发。物理网卡和TAP等虚拟网络设备都可以连接到Linux bridge上。

可以使用`brctl`工具或`iproute2`工具包中的ip命令操作Linux bridge

创建bridge

```bash
brctl addbr br0
```

添加设备到bridge

```bash
brctl addif br0 eth0
```

显示bridge

```bash
brctl show
```

启动bridge

```bash
ip link set dev br0 up
```

停止bridge

```bash
ip link set dev br0 down
```

删除bridge
```bash
brctl delbr br0
```

使用IP命令操作bridge:

创建bridge并启动
```bash
ip link add name br0 type bridge
ip link set dev br0 up
```

先将该端口设置为混杂模式并启动该接口
```bash
ip link set dev eth0 promisc on
ip link set dev eth0 up
```

将接口添加到bridge

```bash
ip link set dev eth0 master br0
```

若要删除网桥，应首先移除它所关联的所有接口，同时关闭接口的混杂模式并关闭接口以将其恢复至原始状态。
```bash
ip link set dev eth0 promisc off
ip link set dev eth0 down
ip link set dev eth0 nomaster
```
删除bridge
```bash
ip link delete br0 type bridge
```

## OVS: Open vSwitch

官网: http://openvswitch.org

OVS是一个产品级别的开源虚拟交换机。相较于Linux bridge, 它提供了各多的功能特性和自动化的编程支持。OVS使用OPENFLOW协议的流表来控制转发逻辑。

一些简单的操作:

创建bridge
```bash
ovs-vsctl add-br ovsbr0
```
查看bridge
```bash
ovs-vsctl show
```
添加Port, 并设置VLAN ID为2:
```bash
ovs-vsctl add-port ovsbr0 tap1 tag=2
```
删除Port
```bash
ovs-vsctl del-port ovsbr0 tap1
```
删除bridge
```bash
ovs-vsctl del-br ovsbr0
```

## network namespace

一般情况下，Linux的网络接口，路由表、协议栈、iptables规则等资源由操作系统的全部进程共享。通过使用netowrk namespace, 可以将这些网络资源隔离开，只由namespace内的进程共享。

namespace示例:

创建namespace
```bash
ip netns add ns1
```
查看namespace
```bash
ip netns list
```
将设备添加进namespace，这样在全局的环境下，该设备不再可见
```bash
ip link set dev tap1 netns ns1
```

查看namespace内设备
```bash
ip netns exec ns1 ip link list
```

可以直接在namespace中执行bash, 再统一对namespace内的设备进行处理
```bash
ip netns exec ns1 bash
```

删除namespace
```bash
ip netns del ns1
```

## veth pair

Virtual Ethernet Pair简称veth pair, 是一对逻辑相连的端口或网络接口。从其中一个端口进入的数据包将从另一端口流出。可以使用veth pair设备来连接Linux bridge或者OVS的bridge，也可以通过veth pair将两个network namespace连接起来。

创建veth pair:
```bash
ip link add dev veth0 type veth peer name veth1
```

查看创建的veth pair:
```bash
[root@compute1 ~]# ip link list
...
29: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether f6:eb:23:3b:f1:5b brd ff:ff:ff:ff:ff:ff
30: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether ba:8d:1c:cf:04:a0 brd ff:ff:ff:ff:ff:ff
```
从输出可以看到对应的接口设备的名称。

以下示例将veth pair的两个接口分别添加到两个namespace中，从而将两个namespace连接起来。
```bash
ip netns add ns1
ip netns add ns2
ip link set veth0 netns ns1
ip link set veth1 netns ns2
ip netns exec ns1 ip link set dev veth0 up
ip netns exec ns2 ip likn set dev veth1 up
```
