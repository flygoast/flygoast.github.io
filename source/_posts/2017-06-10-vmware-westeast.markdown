---
layout: post
title: "VMware vSphere东西向网络防护"
date: 2017-06-10 23:24:22 +0800
comments: true
categories: Virtualization
---
之前的文章<<[VMware vSphere虚拟网络防护](http://www.just4coding.com/blog/2017/05/01/vmvare/)>>介绍了VMware vSphere的网络基础知识和南北向网络安全防护的思路。VMware NSX提供了东西向安全防护的能力。不过VMware NSX的License价格很贵。本文介绍在非NSX环境中，单纯依赖vSphere提供的能力实现东西向网络安全防护的思路。

我们当前一台ESXi上的环境如图:

{% img /images/2017-06-10/1.png %}

<!--more-->

我们在`10.187.160.40`上使用PING访问`10.187.160.41`。由于两IP为同一子网，首先通过ARP广播确定`10.187.160.41`对应的MAC地址，然后由虚拟交换机`vswitch0`将ICMP数据包转发给`10.187.160.41`。这种网络内部的流量我们称为东西向流量。为了进行东西向安全防护，我们可以将么流量引至一个虚拟安全设备，由虚拟安全设备执行安全策略。对于虚拟交换机OpenvSwitch, 我们可以使用OpenFlow协议实现引流，但是vSphere的虚拟交换机不具备这种能力，因而不能使用这种方案。<<[VMware vSphere虚拟网络防护](http://www.just4coding.com/blog/2017/05/01/vmvare/)>>中我们介绍了vSphere虚拟交换机的基础知识。它支持端口组粒度的VLAN设置，我们可以基于不同VLAN之间网络的二层网络隔离性来实现引流。

下面来看具体实现。我们在ESXi主机上新建虚拟交换机`vss_internal`, 在`vss_internal`上创建VLAN TAG分别为`100`和`200`的端口组`vlan100`和`vlan200`。`10.187.160.40`的网卡接入`vlan100`, `10.187.160.41`的网卡接入`vlan200`。我们在新建的虚拟交换机`vss_internal`和原有的虚拟交换机`vswitch0上`各创建一个TAG为`4095`的TRUNK端口组，并开启其`混杂模式`, `MAC地址更改`， `伪传输`等三个安全选项。我们的虚拟安全设备的两个网络接口分别接入其中。在接入`vss_internal`的网络接口上创建两个TAG分别为`100`和`200`的VLAN子接口。在虚拟安全设备上创建一个OVS bridge，并将两个VLAN子接口和接入`vswitch0`的网络接口接入OVS bridge。整体结构如下:

{% img /images/2017-06-10/2.png %}

我们来分析`10.187.160.40`访问`10.187.160.41`的流程。首先`10.187.160.40`的接口发送ARP广播查询`10.187.10.41`的MAC地址。因为两个接口分属不同VLAN，虚拟交换机`vss_internal`不会将ARP数据包转发至`10.187.160.41`的端口。由于虚拟安全设备的`ens192`接口的端口组为trunk, 因而它会收到上述ARP数据包。子接口`vlan100`将VLAN TAG剥去，并发送到`ovs_bridge`，`ovs_bridge`按正常广播逻辑将数据包发送到`vlan200`子接口。`vlan200`子接口给数据包添加VLAN TAG将数据包由`ens192`发出，再次回到虚拟交换机`vss_internal`。此时数据包的VLAN TAG为`200`，因而`vss_internal`将数据包转发至`10.187.160.41`所在接口。`10.187.160.41`发送ARP响应，此时数据包的目的MAC地址为`10.187.160.40`的MAC地址。数据包到达`vss_internal`之后，由于VLAN不同，`vss_internal`并不会将数据包转发至`10.187.160.40`。但是由于`ens192`所在的端口组开启了混杂模式且为trunk模式，`ens192`会收到该数据包。子接口`vlan200`剥去VLAN后，将数据包发送至`ovs_bridge`。`ovs_bridge`会学习到MAC与PORT的对应关系，因而会将数据包发送至`vlan100`。子接口`vlan100`添加VLAN TAG将数据包发送回`vss_internal`。
`vss_internal`再将数据包转发至`10.187.160.40`。ARP请求与响应的流程结束。ICMP的请求与回应的流程与ARP响应流程相似。

ICMP请求包路径如图:

{% img /images/2017-06-10/3.png %}

下面看一下`10.187.160.40`访问ESXi主机外的IP(如网关IP:10.187.160.1)的情形。ARP广播数据包到达`ovs_bridge`后，也会从`ens160`接口发出，`vswitch0`会将广播包由uplink发送至物理交换机。`10.187.160.1`的接口返回ARP响应后，再次从uplink进入`vswitch0`, `vswitch0`将数据包发送至`ens160`。`ens160`将数据包发送到`ovs_bridge`。`ovs_bridge`根据学习到MAC与PORT的对应关系，将数据包发送至`vlan100`。后续流程与上述流程一致。接下来的ICMP请求包流程如下图所示:

{% img /images/2017-06-10/4.png %}

下面来看虚拟安全设备上的具体操作实现。

首先创建两个VLAN子接口:
```bash
ip link add link ens192 name ens192.100 type vlan id 100
ip link add link ens192 name ens192.200 type vlan id 200
```
创建OVS bridge, 并连接相应接口:
```bash
ovs-vsctl add-br ovsbr1
ovs-vsctl add-port ovsbr1 ens192.100
ovs-vsctl add-port ovsbr1 ens192.200
ovs-vsctl add-port ovsbr1 ens160
ip link set up ens192.100
ip link set up ens192.200
```
如果我们需要完成`10.187.160.40`和`10.187.160.41`之间的访问控制，如何实现呢？可以有两种方式:

1. 因为所有流量都要经过OVS bridge, 我们可以直接通过OpenFlow流表来添加ACL规则。
2. 参考OpenStack安全组实现，在OVS bridge和子接间添加一个Linux Bridge，然后由iptables实现访问控制。

首先介绍直接通过OpenFlow流表实现。

默认流表如下:
```plain
[root@localhost ~]# ovs-ofctl dump-flows ovsbr1
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=159533.885s, table=0, n_packets=2612761, n_bytes=196489634, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
```
我们添加一条流将从10.187.160.40访问10.187.160.41的ICMP数据包丢弃:
```bash
ovs-ofctl add-flow ovsbr1 "table=0, priority=10, dl_type=0x0800, nw_proto=0x01, nw_src=10.187.160.40, nw_dst=10.187.160.41, actions=drop”
```
添加后的流表结果如下:
```plain
[root@localhost ~]# ovs-ofctl dump-flows ovsbr1
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=1.840s, table=0, n_packets=1, n_bytes=98, idle_age=1, priority=10,icmp,nw_src=10.187.160.40,nw_dst=10.187.160.41 actions=drop
cookie=0x0, duration=159565.427s, table=0, n_packets=2613181, n_bytes=196523744, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
```
随后再删除该流:
```bash
ovs-ofctl del-flows ovsbr1 “icmp" 
```
我们查看`10.187.160.40`上的测试结果, 添加流时数据包被丢弃了，删除流后，访问再次正常。

{% img /images/2017-06-10/5.png %}

下面介绍基于iptables实现。
首先开启Linux bridge上的iptables:
```bash
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
```
接下来调整网络结构，在OVS bridge与VLAN子接口之间添加Linux Bridge。
```bash
ovs-vsctl del-port ovsbr1 ens192.100
ovs-vsctl del-port ovsbr1 ens192.200
brctl addbr br100
brctl addif br100 r100
brctl addif br100 ens192.100
brctl addbr br200
brctl addif br200 r200
brctl addif br200 ens192.200
ovs-vsctl add-port ovsbr1 o200
ovs-vsctl add-port ovsbr1 o100
ip link set up r200
ip link set up r100
ip link set up o100
ip link set up o200
ip link set up br100
ip link set up br200
```
查看OVS bridge结构:
```plain
[root@localhost ~]# ovs-vsctl show
67c86a45-0c76-4ab9-9f8e-de37545f57fe
    Bridge "ovsbr1" 
        Port "ovsbr1" 
            Interface "ovsbr1" 
                type: internal
        Port "ens160" 
            Interface "ens160" 
        Port "o100" 
            Interface "o100" 
        Port "o200" 
            Interface "o200" 
    ovs_version: “2.4.0" 
```
查看Linux Bridge结构:
```plain
[root@localhost ~]# brctl show
bridge name    bridge id        STP enabled    interfaces
br100        8000.000c29c2a2e2    no        ens192.100
                                            r100
br200        8000.000c29c2a2e2    no        ens192.200
                                            r200
```                                            
调整后结构如图:

{% img /images/2017-06-10/6.png %}

添加访问控制规则, 并查看:
```plain
[root@localhost ~]# iptables -A FORWARD -p icmp -m physdev --physdev-is-bridged --physdev-out ens192.200 -s 10.187.160.40/32 -d 10.187.160.41/32 -j DROP
[root@localhost ~]# iptables -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DROP       icmp --  10.187.160.40        10.187.160.41        PHYSDEV match --physdev-out ens192.200 --physdev-is-bridged

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
接着再清空规则:
```bash
iptables -F
```
从结果上可以看到, 添加规则时数据包被丢弃了，清空规则后访问恢复:

{% img /images/2017-06-10/7.png %}

上述方案基于原来vSphere环境上没有配置VLAN，若是原来ESXi主机上就有多个VLAN，则需要创建多个OVS bridge和在ens160接口上创建子接口来完成VLAN区分，架构如图所示:

{% img /images/2017-06-10/8.png %}

VMware vSphere提供了相应API来操作端口组，修改虚拟机的网络接口所属端口组等，可以利用这些API自动化完成本文上述的网络结构改动与构建。

