---
layout: post
title: "Open vSwitch上OpenFlow流表操作示例"
date: 2017-01-01 23:07:38 +0800
comments: true
categories: OpenStack
---
Open vSwitch提供了OpenFlow命令行工具: ovs-ofctl， 用法及流表语法等细节参考: ovs-ofctl(8)。

本文将通过简单的PING实验来展示OVS上的OpenFlow操作，实验环境宿主机为CentOS7。

创建虚拟交换机:
```bash
ovs-vsctl add-br br-int
```
创建TAP设备:
```bash
ip tuntap add tap0 mode tap
ip tuntap add tap1 mode tap
```
将设备连接到虚拟交换机:
```bash
ovs-vsctl add-port br-int tap0
ovs-vsctl add-port br-int tap1
```

<!--more-->

启动TAP设备:
```bash
ip link set up tap0
ip link set up tap1
```
基于Cirros镜像启动Guest:
```bash
qemu-system-x86_64 cirros-0.3.4-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :0 -device virtio-net-pci,netdev=net0,mac=52:4b:14:90:74:46 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 --daemonize
qemu-system-x86_64 cirros-0.3.4-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :1 -device virtio-net-pci,netdev=net0,mac=2e:f4:42:c1:87:62 -netdev tap,id=net0,ifname=tap1,script=no,downscript=no -name vm1 --daemonize
```
VNC登录Guest，分别配置IP，VM0为192.168.10.100，VM1为192.168.10.101。

在VM0上PING虚拟机VM1，PING成功, 如图:

{% img /images/2017-01-01/1.png %}

在宿主机上查看Port信息, 确认虚拟机对应的PortID:
```bash
[root@compute1 ~]# ovs-ofctl show  br-int
OFPT_FEATURES_REPLY (xid=0x2): dpid:00001237b7e50f43
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
1(tap0): addr:52:4b:14:90:74:46
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
2(tap1): addr:2e:f4:42:c1:87:62
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
LOCAL(br-int): addr:c2:72:e4:46:ba:49
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```
查看流表信息:
```bash
[root@compute1 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=1911.756s, table=0, n_packets=10, n_bytes=756, idle_age=153, priority=0 actions=NORMAL
```
流表0中有一条流，actions为NORMAL，就是按普通交换机处理数据包。

我们添加一条流，将来自VM0对应Port的数据包全部丢弃:
```bash
[root@compute1 ~]# ovs-ofctl add-flow br-int "priority=10,in_port=1,actions=drop" 
[root@compute1 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=2.497s, table=0, n_packets=0, n_bytes=0, idle_age=2, priority=10,in_port=1 actions=drop
cookie=0x0, duration=2302.086s, table=0, n_packets=10, n_bytes=756, idle_age=544, priority=0 actions=NORMAL
```
再次从VM0上PING虚拟机VM1，PING失败，如图:

{% img /images/2017-01-01/2.png %}

再来删除添加的流:
```bash
[root@compute1 ~]# ovs-ofctl del-flows br-int in_port=1
[root@compute1 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=2657.126s, table=0, n_packets=10, n_bytes=756, idle_age=899, priority=0 actions=NORMAL
```
再次进行PING, PING恢复成功:

{% img /images/2017-01-01/3.png %}

接下来我们展示基于OpenFlow流表完成将PING数据包引流至中间网络设备(如防火墙)的操作。

上述PING操作正常的数据包流向如图:

{% img /images/2017-01-01/4.png %}

OVS虚拟交换机基于2层MAC学习机制对数据包进行转发。添加引流逻辑后的数据包流向如图:

{% img /images/2017-01-01/5.png %}

VM0发出的PING请求包到达虚拟交换机后，交换机根据流表将数据包转发至vFW的入口Port。vFW对数据包完成自身过滤逻辑后，将数据包从出口Port送到交换机。交换机再根据MAC地址转发至VM1。VM1做出响应，将数据包送达交换机。交换机根据流表将响应包转发至vFW的入口Port，vFW过滤数据包，将数据包从出口Port发送到交换机，交换机再根据目标MAC地址转发到VM0，完成PING的请求响应逻辑。

下面来看具体操作:

在上述基础上添加两个TAP设备:
```bash
ip tuntap add tap2 mode tap
ip tuntap add tap3 mode tap
```
将设备连接到虚拟交换机:
```bash
ovs-vsctl add-port br-int tap2
ovs-vsctl add-port br-int tap3
```
启动TAP设备:
```bash
ip link set up tap2
ip link set up tap3
```
查看Port信息, 确认Port对应的MAC地址:
```bash
[root@compute1 ~]# ovs-ofctl show br-int
OFPT_FEATURES_REPLY (xid=0x2): dpid:0000c272e446ba49
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
1(tap0): addr:52:4b:14:90:74:46
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
2(tap1): addr:2e:f4:42:c1:87:62
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
3(tap2): addr:0a:9e:a5:55:ef:93
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
4(tap3): addr:f6:94:31:9d:bf:e7
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
LOCAL(br-int): addr:c2:72:e4:46:ba:49
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```

这里我使用标准Linux配置成HUB模式来模拟vFW设备。

使用QEMU启动虚拟机:
```bash
qemu-system-x86_64 CentOS7.img -smp 2,cores=2 -m 2G -vnc :2 -device virtio-net-pci,netdev=net0,mac=0a:9e:a5:55:ef:93 -netdev tap,id=net0,ifname=tap2,script=no,downscript=no -device virtio-net-pci,netdev=net1,mac=f6:94:31:9d:bf:e7 -netdev tap,id=net1,ifname=tap3,script=no,downscript=no -name vFW --daemonize
```

VNC登录vFW后，将Linux配置成Hub模式。
```bash
brctl addbr hub0
brctl addif hub0 eth0
brctl addif hub0 eth1
brctl setageing hub0 0
ip link set up hub0
```

在宿主机上添加流，令PORT3只能做为vFW的入口:
```bash
ovs-ofctl add-flow br-int "cookie=0x10, priority=9, in_port=3, actions=drop" 
```
将vFW的出口Port的ICMP数据包按目标MAC地址转发, 丢弃其他协议数据包:
```bash
ovs-ofctl add-flow br-int "cookie=0x10, priority=8, in_port=4, icmp,dl_dst=52:4b:14:90:74:46,actions=output:1" 
ovs-ofctl add-flow br-int "cookie=0x10, priority=8, in_port=4, icmp,dl_dst=2e:f4:42:c1:87:62,actions=output:2" 
ovs-ofctl add-flow br-int "cookie=0x10, priority=7, in_port=4, actions=drop" 
```
将ICMP数据包引流至vFW的入口Port：
```bash
ovs-ofctl add-flow br-int "cookie=0x10, priority=6, icmp, actions=output:3" 
```

查看此时流表:
```bash
[root@compute1 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
cookie=0x10, duration=8197.204s, table=0, n_packets=218309, n_bytes=9168978, idle_age=8097, priority=9,in_port=3 actions=drop
cookie=0x10, duration=8097.560s, table=0, n_packets=747, n_bytes=31934, idle_age=7732, priority=7,in_port=4 actions=drop
cookie=0x10, duration=8148.358s, table=0, n_packets=165, n_bytes=16170, idle_age=7725, priority=8,icmp,in_port=4,dl_dst=52:4b:14:90:74:46 actions=output:1
cookie=0x10, duration=7731.811s, table=0, n_packets=7, n_bytes=686, idle_age=7725, priority=8,icmp,in_port=4,dl_dst=2e:f4:42:c1:87:62 actions=output:2
cookie=0x10, duration=8060.145s, table=0, n_packets=340, n_bytes=33320, idle_age=7725, priority=6,icmp actions=output:3
cookie=0x0, duration=37097.453s, table=0, n_packets=2755382, n_bytes=116368116, idle_age=7740, priority=0 actions=NORMAL
```

此时，再次从VM0发起PING请求，请求可以成功。

删除转发至VM1的流，开始出现丢包:
```bash
ovs-ofctl del-flows br-int "in_port=4,dl_dst=2e:f4:42:c1:87:62" 
```
再次添加后，请求再次成功。
```bash
ovs-ofctl add-flow br-int "cookie=0x10, priority=8, in_port=4, icmp,dl_dst=2e:f4:42:c1:87:62,actions=output:2"
```

结果截图如下, 可以看其中seq为6-13的请求包没有收到回应。

{% img /images/2017-01-01/6.png %}

本文简单展示了OVS的OpenFlow流表操作，通过自定义流表我们可以完成非常复杂而强大的功能。
