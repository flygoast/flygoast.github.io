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
qemu-system-x86_64 cirros-0.3.4-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :0 -device virtio-net-pci,netdev=net0,mac=32:a7:76:d9:46:2a -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 --daemonize
qemu-system-x86_64 cirros-0.3.4-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :1 -device virtio-net-pci,netdev=net0,mac=3a:98:1b:8e:7f:d0 -netdev tap,id=net0,ifname=tap1,script=no,downscript=no -name vm1 --daemonize
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
1(tap0): addr:32:a7:76:d9:46:2a
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
2(tap1): addr:3a:98:1b:8e:7f:d0
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
LOCAL(br-int): addr:12:37:b7:e5:0f:43
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

本文简单展示了OVS的OpenFlow流表操作，通过自定义流表我们可以完成将数据包引流至中间网络设备(如防火墙)等高级行为, 后续再写文章来说明。

