---
layout: post
title: "基于PVLAN实现VMware vSphere环境DVS东西向防护"
date: 2018-10-10 16:57:45 +0800
comments: true
categories: Virtualization
---
之前的文章<<[VMware vSphere东西向网络防护](/blog/2017/06/10/vmware-westeast/)>>介绍了基于不同VLAN来实现二层网络内虚拟机的东西向隔离防护技术。这种方法在每台ESXi主机上对VLAN ID的管理和分配较为烦琐，本文介绍基于PVLAN实现二层网络内东西向隔离防护的方法。由于VMware vSphere环境中只有`DVS`(Distributed Virtual Switch，分布式虚拟交换机)支持PVLAN，所以这种方法只能使用在DVS中。

PVLAN(`Private VLAN`，专用VLAN)技术是为了解决网络内需要大量隔离而VLAN ID不足的情况。它采用两层VLAN隔离技术，即上行的主VLAN(`Primary VLAN`)和下行的辅助VLAN(`Secondary VLAN`)。对上行设备而言，只可见主VLAN,而不可见辅助VLAN。辅助VLAN包含两种类型:隔离VLAN(`isolated VLAN`)和团体VLAN(`community VLAN`)。支持PVLAN的交换机的端口对应有三种类型: 隔离端口(`isolated port`),团体端口(`community port`),混杂端口(`promiscuous port`)，它们分别对应隔离VLAN，团体VLAN和主VLAN。在隔离VLAN中，隔离端口之间不能通信，隔离端口只能和混杂端口通信。在团体VLAN中，团体端口之间以及团队端口与混杂端口之间都可以通信。

可以看到，隔离端口的特性天生就满足二层网络完全隔离的要求，为了能使他们通信，我们需要虚拟网络设备完成隔离端口间的数据包中继。

我们的测试环境由两台ESXi主机组成，通信双方虚拟机`t1`和`t2`位于同一台ESXi主机，属于同一个二层网络，IP分别为:`10.95.49.150`和`10.95.49.151`, 两台虚拟机的网络接口都接到分布式交换机`DSwitch`的`DPG-original`端口组中。此时拓扑如图:

{% img /images/2018-10-10/1.png %}

<!--more-->


从`t1`访问`t2`, 访问外网都能够成功:

{% img /images/2018-10-10/2.png %}


下面我们通过设置PVLAN实现二层网络隔离。

我们在`DSwitch上`编辑PVLAN。添加一个专用VLAN，设置主VLAN ID为`1000`，辅助VLAN ID为`1001`，类型为隔离VLAN。如图:

{% img /images/2018-10-10/3.png %}


接着，创建分布式端口组:`DPG-isolated`, VLAN类型设置为`专用VLAN`，专用VLAN ID选择`隔离(1000,1001)`，如图:

{% img /images/2018-10-10/4.png %}

再创建分布式端口组:`DPG-promisc`, VLAN类型选择`专用VLAN`，专用VLAN ID选择`混杂(1000,1000)`，如下图所示:

{% img /images/2018-10-10/5.png %}

接着，对`DPG-promisc`进行编辑，需要将安全选项`混杂模式`和`伪传输`设置为接受，否则vSphere DVS将会丢弃符合特定条件的数据包，具体可以参考之前文章<<[VMware vSphere虚拟网络防护](/blog/2017/05/01/vmvare/)>>对于这两个选项的解释, 以及[这篇文章](https://www.virtuallyghetto.com/2013/11/why-is-promiscuous-mode-forged.html)

如下图:
{% img /images/2018-10-10/6.png %}

我们将两个虚拟机的网络接口改到端口组:`DGP-isolated`中，此时拓扑如图:

{% img /images/2018-10-10/7.png %}

我们再次从`t1`访问`t2`。由于隔离端口之间不能通信，因而访问失败:

{% img /images/2018-10-10/8.png %}

因为隔离端口能够与混杂端口通信，我们需要在混杂端口上接入虚拟网络设备，由该设备实现网络转发及防护功能，完成隔离端口之间的通信中继。PVLAN网络中增加了自己的VLAN标识，为了保证能够与原始外界的设备通信，我们需要在虚拟网络设备中将网络数据包转换为没有设置PVLAN之前的状态，因而我们需要虚拟网络设备的一个网络接口接入原有的端口组中, 整体架构如图:

{% img /images/2018-10-10/9.png %}

下面我们看具体操作。

我们创建一台具备两个网络接口的虚拟机`VNF`, 第一个接口接入端口组`DPG-promisc`, 第二个接口接入`t1`,`t2`原来的端口组:`DPG-original`。因为`t1`,`t2`访问外部的流量需要经由`VNF`虚拟机的`eth1`接口发出，因而`DPG-original`也需要将端口组的`混杂模式`和`伪传输`两个安全选项设置为`接受`。此时拓扑如图:

{% img /images/2018-10-10/10.png %}

我们在VNF设备内使用OpenvSwitch实现转发逻辑。

首先，我们需要将两个网络接口添加进`br0`中

```bash
ovs-vsctl add-br br0
ovs-vsctl add-port br0 eth0
ovs-vsctl add-port br0 eth1
```

查看ofport情况:
```plain
[root@localhost ~]# ovs-ofctl show br0
OFPT_FEATURES_REPLY (xid=0x2): dpid:0000005056a023f3
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(eth0): addr:00:50:56:a0:48:ee
     config:     0
     state:      0
     current:    10GB-FD COPPER
     advertised: COPPER
     supported:  1GB-FD 10GB-FD COPPER
     speed: 10000 Mbps now, 10000 Mbps max
 2(eth1): addr:00:50:56:a0:23:f3
     config:     0
     state:      0
     current:    10GB-FD COPPER
     advertised: COPPER
     supported:  1GB-FD 10GB-FD COPPER
     speed: 10000 Mbps now, 10000 Mbps max
 LOCAL(br0): addr:00:50:56:a0:23:f3
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```

开启`STP`, 避免形成二层环路，造成网络中断:
```bash
ovs-vsctl set bridge br0 stp_enable=true
```

在OVS中，在如下这种流表条件下:
```plain
in_port=1 actions=output:1
```
从端口`1`中接收的数据包是不能由端口`1`发送出去的，需要使用特殊的端口`IN_PORT`来处理。

`t1`访问`t2`的流量需要由`br0`的端口`eth0`进入并再次由它发出，而访问外部的设备时需要由`eth1`端口发出，因而需要对源MAC地址进行学习来判断应该从哪个端口发出。

我们本身共使用三张流表:

* 表0: 用于处理ARP数据包，通过ARP学习源MAC地址与PORT的对应关系，将学习到的流保存到表10中, 并将其他协议数据包交由表10和表11处理。
* 表10: 存储学习到的流，当数据包匹配到表10中的流时，它将学习到的PORT(即应从该PORT发出该数据包)保存到`REG0`中。
* 表11: 根据`REG0`中存储的PORT信息，修改入端口信息，保证出入端口不相同，从而可以成功发送。

添加表0的流:
```bash
ovs-ofctl add-flow br0 "table=0, priority=10,arp,in_port=1 actions=learn(table=10,hard_timeout=300,priority=10,NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),output:2,output:IN_PORT"
ovs-ofctl add-flow br0 "table=0, priority=10,arp,in_port=2 actions=learn(table=10,hard_timeout=300,priority=10,NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),output:1"
ovs-ofctl add-flow br0 "table=0, priority=9, actions=resubmit(,10), resubmit(,11)"
```

添加表11的流:
对于从`eth1`进入的数据包我们直接从`eth0`转发走:
```bash
ovs-ofctl add-flow br0 "table=11,priority=11,in_port=2 actions=output:1”
```
根据`REG0`中保存的出端口信息，修改入端口并由相应出端口转发:
```bash
ovs-ofctl add-flow br0 "table=11,priority=10,reg0=1,actions=set_field:2->in_port_oxm, output:1"
ovs-ofctl add-flow br0 "table=11,priority=10,reg0=2,actions=set_field:1->in_port_oxm, output:2”
```
没有匹配到学习到的MAC，则从两个端口都转发:
```bash
ovs-ofctl add-flow br0 "table=11,priority=9,in_port=1 actions=output:2,output:IN_PORT"
```

此时，再次从`t1`访问`t2`,访问成功:

{% img /images/2018-10-10/11.png %}

在`VNF`上的`eth0`接口上抓包可以看到数据通过了`VNF`进行转发:

{% img /images/2018-10-10/12.png %}

再次访问外网，访问也成功:

{% img /images/2018-10-10/13.png %}

在`VNF`上的`eth1`接口上的抓包结果显示数据包通过了`eth1`接口转发:

{% img /images/2018-10-10/14.png %}

整个`DVS`可以只创建一台VNF虚拟机完成所有二层网络整体流量的转发和过滤，但这样也会导致本可以在ESXi主机内完成转发的数据包需要先发送到VNF所在ESXi主机上完成过滤和转发再回到原有ESXi主机内，这给物理网络增加了额外负担。

除了这种形式外，也可以在每台ESXi主机上创建一台VNF，该VNF负责该ESXi主机上的虚拟机的流量过滤，这种情形下需要给不同的ESXi主机分配不一样的PVLAN ID，整体架构如图:

{% img /images/2018-10-10/15.png %}

但是由于`DPG-original`端口组需要设置为`混杂模式`，处于混杂模式的端口组中的网络接口会接收到穿过虚拟交换机的所有网络流量，因而也会导致许多网络数据包在不同的ESXi主机之间进行没有必要的传输，造成网络吞吐的衰退。

之所以需要设置为`混杂模式`，是由于VMware vSphere的虚拟交换机(VSS/DVS)不支持`Mac-Learning`机制，因为vSphere平台本身就知道网络接口的MAC地址。然而在虚拟化嵌套或者我们这种由虚拟网络设备中转二层数据包的情况下，需要由一个网络接口发送大量的不同MAC地址的数据包, vSphere平台并不知道该接口会发送哪些MAC地址的数据包。VMware官方为了解决这个问题，开发了一个dvfilter-maclearn模块，具体可参考[文章](https://www.virtuallyghetto.com/2014/08/new-vmware-fling-to-improve-networkcpu-performance-when-using-promiscuous-mode-for-nested-esxi.html)。

`dvfilter-maclearn`需要安装在ESXi主机上:
```plain
[root@ESXi-2:~] esxcli software vib install -v /vmware-esx-dvfilter-maclearn-1.0.vib -f
Installation Result
   Message: Operation finished successfully.
   Reboot Required: false
   VIBs Installed: VMware_bootbank_vmware-esx-dvfilter-maclearn_1.00-1.00
   VIBs Removed:
   VIBs Skipped:
```

安装完成后，执行下述命令可以看到`dvfilter-maclearn`的安装情况:
```bash
/sbin/summarize-dvfilter
```

结果如图:

{% img /images/2018-10-10/16.png %}


为了使这个`dvFilter`生效，需要针对相应虚拟机的网卡进行过滤设置。我们需要在相应虚拟机的VMX配置文件中添加:
```plain
ethernet#.filter4.name=dvfilter-maclearn
ethernet#.filter4.onFailure=failOpen
```

`#`为网卡的顺序号，第一个网卡为`0`。

也可以通过VCenter进行配置，如图:

{% img /images/2018-10-10/17.png %}

生效后再次执行/sbin/summarize-dvfilter，查看dvfilter加载情况，可以看到`dvfilter-maclearn`已经生效，在虚拟机相应网卡上抓包可以看到数据包有明显减少。

{% img /images/2018-10-10/18.png %}

`dvfilter-maclearn`尽管可以提高部分性能还是存在一些问题，比如学习到的MAC地址永不过期等等，VMware后续又发布了一个更为完善的方案，具体可参考[文章](https://www.virtuallyghetto.com/2017/04/esxi-learnswitch-enhancement-to-the-esxi-mac-learn-dvfilter.html), 而在VSphere6.7中，`Mac Learning`已经是DVS的原生特性了，混杂模式也不需要再开启，具体参考[文章](https://www.virtuallyghetto.com/2018/04/native-mac-learning-in-vsphere-6-7-removes-the-need-for-promiscuous-mode-for-nested-esxi.html), 这里不再详述。
