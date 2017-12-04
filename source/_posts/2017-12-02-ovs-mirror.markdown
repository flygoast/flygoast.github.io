---
layout: post
title: "OpenvSwitch端口镜像"
date: 2017-12-02 22:26:35 +0800
comments: true
categories: Virtualization 
---
之前的文章<<[VMware vSphere平台端口镜像](http://www.just4coding.com/blog/2017/11/07/port-mirroring/)>>介绍了`VMware vSphere`环境下虚拟交换机的端口镜像机制，本文来介绍在`OpenvSwitch`环境中如何实现端口镜像。

OVS上实现端口镜像的基本流程如下:

* 创建`mirror`，在`mirror`中指定镜像数据源及镜像目的地
* 将创建的`mirror`应用到`bridge`中

镜像数据源可以通过下面几个选项来指定:

* `select_all`: 布尔值，设置为`true`时，进出该`mirror`所生效的`bridge`上的每个数据包都将被镜像
* `select_dst_port`: 从该`port`离开虚拟交换机的数据包将会被镜像，从Guest角度看是Guest网络接口的流入方向
* `select_src_port`: 从该`port`进入虚拟交换机的数据包将会被镜像，从Guest角度看是Guest网络接口的流出方向
* `select_vlan`: 指定特定VLAN做为数据源，整个VLAN的数据包都会镜像到目的地

镜像目的地可以用下面选项来指定:

* `output_port`: 将数据包镜像到特定的`port`
* `output_vlan`: 将数据包镜像到指定VLAN, 原始数据的VLAN tag会被剥掉。若镜像多个VLAN到同一个VLAN，没有办法区分镜像后的数据包来源于哪个VLAN。

<!--more-->

下面我们通过实例来说明OVS上的镜像机制。我们的第一个实验拓朴结构如下图，我们将流入`tap1`网络接口的数据包镜像到`tap3`中:

{% img /images/2017-12-02/1.png %}

首先构造环境:
```bash
ovs-vsctl add-br br0
ovs-vsctl add-port br0 tap1 -- set interface tap1 type=internal
ovs-vsctl add-port br0 tap2 -- set interface tap2 type=internal
ovs-vsctl add-port br0 tap3 -- set interface tap3 type=internal
ip netns add ns1
ip netns add ns2
ip netns add ns3
ip link set dev tap1 netns ns1
ip link set dev tap2 netns ns2
ip link set dev tap3 netns ns3
ip netns exec ns1 ip addr add 10.10.10.11/24 dev tap1
ip netns exec ns1 ip link set up tap1
ip netns exec ns2 ip addr add 10.10.10.12/24 dev tap2
ip netns exec ns2 ip link set up tap2
ip netns exec ns3 ip link set up tap3
```

我们从`ns1`中PING `ns2`的IP:
```plain
[root@centos3 vagrant]# ip netns exec ns1 ping 10.10.10.12 -c 2
PING 10.10.10.12 (10.10.10.12) 56(84) bytes of data.
64 bytes from 10.10.10.12: icmp_seq=1 ttl=64 time=0.060 ms
64 bytes from 10.10.10.12: icmp_seq=2 ttl=64 time=0.106 ms

--- 10.10.10.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.060/0.083/0.106/0.023 ms
```
在`ns3`中运行`tcpdump`观察是否能收到数据包, 可以看到此时`ns3`的`tap3`并不会收到`tap1`访问`tap2`的数据包:
```plain
[root@centos3 vagrant]# ip netns exec ns3 tcpdump -i tap3 -e -nn icmp or arp
tcpdump: WARNING: tap3: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap3, link-type EN10MB (Ethernet), capture size 65535 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```
接下来我们创建了相应的`mirror`, 并将其应用到`br0`上:
```bash
ovs-vsctl -- --id=@tap1 get port tap1  \
          -- --id=@tap3 get port tap3  \
          -- --id=@m create mirror name=m0 select_dst_port=@tap1 output_port=@tap3 \
          -- set bridge br0 mirrors=@m
```
此时查看`OVS`上的`mirror`:
```plain
[root@centos3 vagrant]# ovs-vsctl list mirror
_uuid               : 98b89127-cf94-4926-8d5f-76145154b03c
external_ids        : {}
name                : "m0"
output_port         : 1dcde312-e33d-439f-b646-9db92416a586
output_vlan         : []
select_all          : false
select_dst_port     : [16db4055-5b5b-411d-8e98-87813ba14eff]
select_src_port     : []
select_vlan         : []
statistics          : {tx_bytes=0, tx_packets=0}
```
再次进行PING实验:
```plain
[root@centos3 vagrant]# ip netns exec ns1 ping 10.10.10.12 -c 2
PING 10.10.10.12 (10.10.10.12) 56(84) bytes of data.
64 bytes from 10.10.10.12: icmp_seq=1 ttl=64 time=0.274 ms
64 bytes from 10.10.10.12: icmp_seq=2 ttl=64 time=0.341 ms

--- 10.10.10.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.274/0.307/0.341/0.037 ms
```
在`ns3`上抓包可以看到成功获得`tap2`回应`tap1`的ICMP响应数据包:
```plain
[root@centos3 vagrant]# ip netns exec ns3 tcpdump -i tap3 -e -nn icmp or arp
tcpdump: WARNING: tap3: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap3, link-type EN10MB (Ethernet), capture size 65535 bytes
23:22:11.919448 26:1e:74:67:6c:cc > 16:fe:12:ad:f0:4f, ethertype IPv4 (0x0800), length 98: 10.10.10.12 > 10.10.10.11: ICMP echo reply, id 4411, seq 1, length 64
23:22:12.919823 26:1e:74:67:6c:cc > 16:fe:12:ad:f0:4f, ethertype IPv4 (0x0800), length 98: 10.10.10.12 > 10.10.10.11: ICMP echo reply, id 4411, seq 2, length 64
23:22:16.929503 26:1e:74:67:6c:cc > 16:fe:12:ad:f0:4f, ethertype ARP (0x0806), length 42: Request who-has 10.10.10.11 tell 10.10.10.12, length 28
^C
3 packets captured
3 packets received by filter
0 packets dropped by kernel
```

这种方式对应了`Cisco`的`SPAN`方式, 下面我们来实验`RSPAN`方式。我们将拓朴修改为如下图所示，我们首先在`br0`上将`tap1`的数据包镜像到物定VLAN: `111`, 在`br1`上再从VLAN: `111`中将数据包镜像到`tap4`:

{% img /images/2017-12-02/2.png %}

准备拓朴:
```bash
ip link add p0 type veth peer name p1
ovs-vsctl add-port br0 p0
ovs-vsctl add-port br1 p1
ovs-vsctl add-port br1 tap4 -- set interface tap4 type=internal
ip netns add ns4
ip link set tap4 netns ns4
ip netns exec ns4 ip link set up tap4
```

关闭VLAN:`111`的MAC学习功能，避免影响正常网络转发:
```bash
ovs-vsctl set bridge br0 flood_vlans=111
ovs-vsctl set bridge br1 flood_vlans=111
```

首先创建一个`mirror`将`tap1`的数据包镜像到VLAN:`111`:
```bash
ovs-vsctl -- --id=@tap1 get port tap1  \
          -- --id=@m create mirror name=m1 select_src_port=@tap1 output_vlan=111 \
          -- set bridge br0 mirrors=@m
```
查看`OVS`上的`mirror`:
```plain
[root@centos3 vagrant]# ovs-vsctl list mirror
_uuid               : fd39fdb2-ab69-47c1-bde9-01b7b40dd4d3
external_ids        : {}
name                : "m1"
output_port         : []
output_vlan         : 111
select_all          : false
select_dst_port     : []
select_src_port     : [16db4055-5b5b-411d-8e98-87813ba14eff]
select_vlan         : []
statistics          : {tx_bytes=13426, tx_packets=157}
```
再次从`tap1`发送PING包到`tap2`, 我们在`tap4`上抓包, 可以看到`tap4`上可以收到镜像的数据包:
```plain
[root@centos3 vagrant]# ip netns exec ns4 tcpdump -i tap4 -e -nn icmp or arp
tcpdump: WARNING: tap4: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap4, link-type EN10MB (Ethernet), capture size 65535 bytes
14:45:58.762260 16:fe:12:ad:f0:4f > 26:1e:74:67:6c:cc, ethertype 802.1Q (0x8100), length 102: vlan 111, p 0, ethertype IPv4, 10.10.10.11 > 10.10.10.12: ICMP echo request, id 9722, seq 1, length 64
14:45:59.806203 16:fe:12:ad:f0:4f > 26:1e:74:67:6c:cc, ethertype 802.1Q (0x8100), length 102: vlan 111, p 0, ethertype IPv4, 10.10.10.11 > 10.10.10.12: ICMP echo request, id 9722, seq 2, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```
此时，`tap4`为`trunk`模式，所有VLAN的数据包都可以收到。我们将`tap4`的`tag`设置为`111`:
```bash
ovs-vsctl set port tap4 tap=111
```
此时，再次重复PING访问，`tap4`上依然可以收到镜像的数据包, 只不过VLAN TAG已经被剥除:
```plain
[root@centos3 vagrant]# ip netns exec ns4 tcpdump -i tap4 -e -nn icmp or arp
tcpdump: WARNING: tap4: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap4, link-type EN10MB (Ethernet), capture size 65535 bytes
14:47:24.060144 16:fe:12:ad:f0:4f > 26:1e:74:67:6c:cc, ethertype IPv4 (0x0800), length 98: 10.10.10.11 > 10.10.10.12: ICMP echo request, id 9816, seq 1, length 64
14:47:25.117508 16:fe:12:ad:f0:4f > 26:1e:74:67:6c:cc, ethertype IPv4 (0x0800), length 98: 10.10.10.11 > 10.10.10.12: ICMP echo request, id 9816, seq 2, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```
我们再次将`tag`设置为`110`，再次从`tap4`抓包则收不到数据包。

我们可以添加另一个镜像规则，将镜像VLAN的数据包镜像到`tap4`:
```bash
ovs-vsctl -- --id=@tap4 get port tap4  \
          -- --id=@m create mirror name=m2 select_vlan=111 select_all=true output_port=@tap4 \
          -- add bridge br1 mirrors @m
```
查看OVS上的`mirror`:
```plain
[root@centos3 vagrant]# ovs-vsctl list mirror
_uuid               : e758d8ae-408d-4e4f-a1c4-eaa8dd99bce5
external_ids        : {}
name                : "m2"
output_port         : e6a7d9b6-1032-4f9a-84f1-54b69f6db315
output_vlan         : []
select_all          : true
select_dst_port     : []
select_src_port     : []
select_vlan         : [111]
statistics          : {tx_bytes=0, tx_packets=0}

_uuid               : fd39fdb2-ab69-47c1-bde9-01b7b40dd4d3
external_ids        : {}
name                : "m1"
output_port         : []
output_vlan         : 111
select_all          : false
select_dst_port     : []
select_src_port     : [16db4055-5b5b-411d-8e98-87813ba14eff]
select_vlan         : []
statistics          : {tx_bytes=14378, tx_packets=169}
```

在`tap4`上抓包, 得到的数据包已经被剥掉VLAN TAG:
```plain
[root@centos3 vagrant]# ip netns exec ns4 tcpdump -i tap4 -e -nn icmp or arp
tcpdump: WARNING: tap4: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap4, link-type EN10MB (Ethernet), capture size 65535 bytes
14:50:37.842958 16:fe:12:ad:f0:4f > 26:1e:74:67:6c:cc, ethertype IPv4 (0x0800), length 98: 10.10.10.11 > 10.10.10.12: ICMP echo request, id 10044, seq 1, length 64
14:50:38.904391 16:fe:12:ad:f0:4f > 26:1e:74:67:6c:cc, ethertype IPv4 (0x0800), length 98: 10.10.10.11 > 10.10.10.12: ICMP echo request, id 10044, seq 2, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```
使用镜像到物定VLAN的方式会导致原始的VLAN TAG信息丢失，如果要镜像多个VLAN的数据到同一目的地则会造成混乱。这种场景下，最好将数据包镜像到一个GRE port，基于这种方式可以实现`ERSPAN`模式。

清除`mirror`设置:
```bash
ovs-vsctl clear bridge br0 mirrors
ovs-vsctl clear bridge br1 mirrors
```

添加一个GRE端口:
```bash
ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:key=0x1111 options:remote_ip=10.95.30.43
```

创建`mirror`将`tap1`的数据包镜像至GRE端口:
```bash
ovs-vsctl -- --id=@tap1 get port tap1  \
          -- --id=@gre0 get port gre0  \
          -- --id=@m create mirror name=m3 select_src_port=@tap1 output_port=@gre0 \
          -- set bridge br0 mirrors=@m
```
查看`mirror`:
```plain
[root@centos3 vagrant]# ovs-vsctl list mirror
_uuid               : 3c63e590-d42c-454a-a7f9-baa3e0ebb77f
external_ids        : {}
name                : "m3"
output_port         : e71af801-289f-466b-9f66-d1e5cf396233
output_vlan         : []
select_all          : false
select_dst_port     : []
select_src_port     : [16db4055-5b5b-411d-8e98-87813ba14eff]
select_vlan         : []
statistics          : {tx_bytes=0, tx_packets=0}
```
我们在外网出口`eth0`上抓包，可以看到，GRE数据包已经发送:
```plain
[root@centos3 vagrant]# tcpdump -ieth0 -nn -e proto gre
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
15:46:50.236791 08:00:27:6b:57:88 > 52:54:00:12:35:02, ethertype IPv4 (0x0800), length 140: 10.0.2.15 > 10.95.30.43: GREv0, key=0x1111, proto TEB (0x6558), length 106: 86:5e:7a:ba:4a:5f > e2:ee:06:10:cb:70, ethertype IPv4 (0x0800), length 98: 10.10.10.11 > 10.10.10.12: ICMP echo request, id 1941, seq 1, length 64
15:46:51.237935 08:00:27:6b:57:88 > 52:54:00:12:35:02, ethertype IPv4 (0x0800), length 140: 10.0.2.15 > 10.95.30.43: GREv0, key=0x1111, proto TEB (0x6558), length 106: 86:5e:7a:ba:4a:5f > e2:ee:06:10:cb:70, ethertype IPv4 (0x0800), length 98: 10.10.10.11 > 10.10.10.12: ICMP echo request, id 1941, seq 2, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```
