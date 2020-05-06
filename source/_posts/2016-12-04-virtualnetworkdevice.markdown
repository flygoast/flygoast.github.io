---
layout: post
title: "Linux虚拟网络设备"
date: 2016-12-04 18:21:36 +0800
comments: true
tags:
- interface
- network
- tap
categories: Network
---
更新(2020-05-06):
Redhat在2018年发表了一篇关于Linux虚拟网络设备的文章，内容比本文要更详细, 可以参考:
https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking

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

OVS是一个产品级别的开源虚拟交换机。相较于Linux bridge, 它提供了各多的功能特性和自动化的编程支持。OVS中涉及如下虚拟网络设备:

* bridge: 相较与Linux bridge，它基于OPENFLOW协议的流表来控制二层转发逻辑。
* port: 网络设备在OVS bridge上的挂接点。
* interface: 挂接到OVS端口的网络接口，可能是OVS创建的类型为internal的虚拟网卡,也可能是操作系统上的物理网卡或TUN/TAP等虚拟网络设备

其中, Port有几种类型:

* normal: 表示Port上挂接的是操作系统的网络接口(物理网卡或者TUN/TAP设备)
* internal: Port被设置成internal类型时，OVS会自动创建一个虚拟网卡挂接到该Port
* patch: patch类型的Port用于连接两个OVS bridge, 从而在两个bridge之间转发数据, 它总是成对出现，可以理解为一根虚拟电缆
* tunnel: 支持使用GRE和VXLAN等隧道技术与远程端口通信

简单的操作示例:

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
添加Port，并设置类型为internal
```bash
ovs-vsctl add-port ovsbr0 p0 -- set interface p0 type=internal
```
创建patch类型的port:
```bash
ovs-vsctl add-port ovsbr0 p1 -- set interface p1 type=patch -- set interface p1 options:peer=p2
ovs-vsctl add-port ovsbr1 p2 -- set interface p2 type=patch -- set interface p2 options:peer=p1
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

## MACVLAN

一般情况下，网卡只有一个MAC地址。然而，有些场景下需要给一个网卡设置多个MAC地址。Linux通过MACVLAN技术在一个物理网卡上创建多个MACVLAN虚拟设备，每个设备有着不同的MAC地址。当物理网卡收到数据包时，MACVLAN driver根据数据包MAC地址将数据包交由匹配的虚拟网卡处理。使用MACVLAN可以替代使用bridge来连接物理网卡和虚拟网络设备。

创建MACVLAN设备并自动分配MAC地址:
```bash
ip link add link ens160 name mac0 type macvlan
```
查看生成的设备信息:
```bash
[root@localhost ~]# ip link show mac0
60: mac0@ens160: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT
    link/ether 4e:d0:8f:d3:e1:1e brd ff:ff:ff:ff:ff:ff
```
也可以指定MAC地址创建设备:
```bash
ip link add link ens160 name mac1 address 52:22:33:44:55:66 type macvlan
```
指定MAC地址需要注意:第一字节的第0位应为0，表示单播MAC地址，1表示多播MAC地址。第1位应为”1”, 这表示，这个MAC地址是本地地址。

删除MACVLAN设备
```bash
ip link del mac0
```

## MACVTAP

虚拟化中一般使用TAP和bridge来组建虚拟网络，但这样组网结构会稍显复杂。Linux上的MACTAP设备可以简化这种结构。MACVTAP设备集成了MACVLAN和TAP设备二者的特性。它也可以基于一个物理网卡创建多个MAC地址不同的虚拟网卡，同时虚拟网卡收到的包不再交给内核协议栈，而是通过TAP设备的文件描述符传递到用户态进程。

MACVTAP和MACVLAN类型的设备有4种工作模式:

* VEPA(Virtual Ethernet Port Aggregator): 同一物理网卡下的虚拟网卡之间的流量要先发送到外部交换机，再由交换机发送回物理网卡。但是外部交换机应支持并配置成hairpin模式
* Bridge: 与Linux bridge类似，同一物理网卡下的虚拟网卡之间可以直接交换数据包，无需外部交换机
* Private: 同一物理网卡下的虚拟网卡之间无法连通，无论外部交换机是否支持hairpin模式
* Passthru: MACVALN和MACVTAP的数据处理逻辑被跳过，相当于虚拟网卡直接连接物理网卡。这种模式的好处是虚拟网卡可以修改MAC地址或其他接口参数

一个MACVTAP设备被创建后，会生成一个字符设备文件在/dev/下，一般为”/dev/tapNN”， NN是MACVTAP设备的接口索引值。
创建MACVTAP设备并设置mode为bridge:
```bash
ip link add link ens160 macvtap0 type macvtap mode bridge
```
启动设备:
```bash
ip link set dev macvtap0 up
```
查看MACVTAP设备:
```bash
[root@localhost ~]# ip link show macvtap0
61: macvtap0@ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT qlen 500
    link/ether da:fc:86:c8:b7:52 brd ff:ff:ff:ff:ff:ff
```
在/dev下可以看到生成的字符设备文件:
```bash
[root@localhost ~]# ls -l /dev/tap61
crw-------. 1 root root 248, 1 Dec  8 10:57 /dev/tap61
```
删除设备:
```bash
ip link del macvtap0
```
