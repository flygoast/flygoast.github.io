---
layout: post
title: "OpenStack私有网络二层接入虚拟防火墙"
date: 2018-08-08 19:47:06 +0800
comments: true
categories: OpenStack
---
默认情况下，在OpenStack上私有网络上开启的虚拟机接口必须配置IP，否则网络包无法到达该接口。而之前的文章<<[OpenStack私有网络旁路部署虚拟防火墙](/blog/2018/08/08/openstack-vfw/)>>最后提到某些厂商的VFW只支持二层透明接入，那么我们怎样在OpenStack环境上跑通这样的VFW实例呢？本文来通过实例说明。

上篇文章的示例网络拓扑如下: 

{% img /images/2018-08-08/21.png %}

<!--more-->

VFW虚拟机接口与vRouter之间依赖IP协议转发数据包。由于VFW需要二层接入，因而不能够在防火墙接口上配置IP，我们可以在VFW前添加一个伪造的vRouter(后文称为Fake vRouter)，将IP配置在该vRouter上，vRouter与VFW之间实现二层转发，网络拓扑变为:

{% img /images/2018-08-08/22.png %}

原来拓扑对应的实际结构如图:

{% img /images/2018-08-08/23.png %}

新拓扑对应的实际网络结构如图:

{% img /images/2018-08-08/24.png %}

下面我们基于上篇文章的环境继续操作。我的环境为OpenStack ALL IN ONE环境，VFW虚拟机和vRouter都在同一台机器上。

我们首先给VFW虚拟机添加一个接口，原接口作入接口，新接口作为出接口。

设置OpenStack相应的环境变量后，查看`vfw`网络ID:
```plain
[root@aio ~(keystone_admin)]# neutron net-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+---------+----------------------------------+-------------------------------------------------------+
| id                                   | name    | tenant_id                        | subnets                                               |
+--------------------------------------+---------+----------------------------------+-------------------------------------------------------+
| 16b1e02f-216f-4e38-859c-a4cb87239106 | private | 045e1b8ee842483db6819f6ff1c966f7 | 17858aee-b868-4719-bf79-f225f2120209 10.0.0.0/24      |
| 6b930b9b-3a93-4300-b4d9-6312ed36432c | public  | 4d885395e1b04f97a61f0288ef41e307 | 5cf81ad0-d266-4b42-9a83-2b9885f32ece 172.24.4.0/24    |
| a18c3985-535e-4d18-a49e-900912de5086 | vfw     | 4d885395e1b04f97a61f0288ef41e307 | fbcf8e7a-89c8-4c74-a6ef-5178f8ab0ebd 192.168.100.0/24 |
| c1ed4eb2-c45b-44e8-ac26-efd592c11860 | demo    | 4d885395e1b04f97a61f0288ef41e307 | 8c09028b-e730-457b-917b-5102cefcbe6f 10.10.10.0/24    |
+--------------------------------------+---------+----------------------------------+-------------------------------------------------------+
```

在`vfw`子网中创建一个新接口:
```plain
[root@aio ~(keystone_admin)]# neutron port-create a18c3985-535e-4d18-a49e-900912de5086
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Created a new port:
+-----------------------+--------------------------------------------------------------------------------------+
| Field                 | Value                                                                                |
+-----------------------+--------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                 |
| allowed_address_pairs |                                                                                      |
| binding:host_id       |                                                                                      |
| binding:profile       | {}                                                                                   |
| binding:vif_details   | {}                                                                                   |
| binding:vif_type      | unbound                                                                              |
| binding:vnic_type     | normal                                                                               |
| created_at            | 2018-08-08T09:40:21Z                                                                 |
| description           |                                                                                      |
| device_id             |                                                                                      |
| device_owner          |                                                                                      |
| extra_dhcp_opts       |                                                                                      |
| fixed_ips             | {"subnet_id": "fbcf8e7a-89c8-4c74-a6ef-5178f8ab0ebd", "ip_address": "192.168.100.8"} |
| id                    | f97f143d-9121-4a52-a898-850633cac943                                                 |
| mac_address           | fa:16:3e:ee:d0:27                                                                    |
| name                  |                                                                                      |
| network_id            | a18c3985-535e-4d18-a49e-900912de5086                                                 |
| port_security_enabled | True                                                                                 |
| project_id            | 4d885395e1b04f97a61f0288ef41e307                                                     |
| revision_number       | 3                                                                                    |
| security_groups       | 15915a82-919a-41b4-96d3-3cac6652079d                                                 |
| status                | DOWN                                                                                 |
| tags                  |                                                                                      |
| tenant_id             | 4d885395e1b04f97a61f0288ef41e307                                                     |
| updated_at            | 2018-08-08T09:40:22Z                                                                 |
+-----------------------+--------------------------------------------------------------------------------------+
```

将这个接口添加到虚拟机VFW上:
```plain
[root@aio ~(keystone_admin)]# nova interface-attach --port-id f97f143d-9121-4a52-a898-850633cac943 vfw
[root@aio ~(keystone_admin)]# nova list
+--------------------------------------+------+--------+------------+-------------+----------------------------------+
| ID                                   | Name | Status | Task State | Power State | Networks                         |
+--------------------------------------+------+--------+------------+-------------+----------------------------------+
| c7da7c30-fd95-4cb2-b412-45c911e320e5 | app1 | ACTIVE | -          | Running     | demo=10.10.10.9, 172.24.4.4      |
| 2e0deaeb-d140-4d52-8289-efdc05267b52 | vfw  | ACTIVE | -          | Running     | vfw=192.168.100.6, 192.168.100.8 |
+--------------------------------------+------+--------+------------+-------------+----------------------------------+
```

登录到VFW虚拟机上，将虚拟机的两个网络接口串接起来模拟二层防火墙:
```bash
brctl addbr br0
brctl addif eth0
brctl addif eth1
brctl setageing br0 0
ip link set up dev eth0
ip link set up dev eth1
```

查看串接后的网桥:
```
[root@vfw ~]# brctl show
bridge name    bridge id            STP enabled    interfaces
br0            8000.fa163e73c1de    no             eth0
                                                   eth1
```

接着回到OpenStack宿主机来操作, 找到VFW虚拟机的接口ID:
```plain
[root@aio ~(keystone_admin)]# neutron port-list |grep 192.168.100.6
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
| 2cdddce0-6533-4d2d-9214-1214e0b2375b |      | 4d885395e1b04f97a61f0288ef41e307 | fa:16:3e:73:c1:de | {"subnet_id": "fbcf8e7a-89c8-4c74-a6ef-5178f8ab0ebd", "ip_address": "192.168.100.6"} |
```

查看VFW虚拟机接口在`br-int`上的VLAN tag ID, 可以看到VLAN TAG为`4`:
```plain
[root@aio ~]# ovs-vsctl show
aef92bf7-dcdc-4b6f-9add-d30f929940ff
    ...
    Bridge br-int
        ...
        Port "qvo2cdddce0-65"
            tag: 4
            Interface "qvo2cdddce0-65"
        ...

    ovs_version: "2.9.0"
```

在`br-int`上创建一个虚拟接口`vfw`, 并将其VLAN TAG设置为`4`:
```bash
ovs-vsctl add-port br-int vfw -- set interface vfw type=internal
ovs-vsctl set port vfw tag=4
```

创建连接VFW虚拟机出入接口的网桥:
```bash
ovs-vsctl add-br vfw-in
ovs-vsctl add-br vfw-out
```

创建Fake vRouter的net namespace:
```bash
ip netns add qrouter-vfw
```

将`br-int`上的`vfw`, `vfw-in`, `vfw-out`三个接口都放入`qrouter-vfw`中:
```bash
ip link set dev vfw netns qrouter-vfw
ip link set dev vfw-in netns qrouter-vfw
ip link set dev vfw-out netns qrouter-vfw
```

再将VFW虚拟机的两个接口移到新建的两个网桥上:
```bash
brctl delif qbr2cdddce0-65 tap2cdddce0-65
ovs-vsctl add-port vfw-in tap2cdddce0-65

brctl delif qbrf97f143d-91 tapf97f143d-91
ovs-vsctl add-port vfw-out tapf97f143d-91
```

接下来，我们进入到`qrouter-vfw`中:
```bash
ip netns exec qrouter-vfw bash
```

此时的网络接口情况如下:
```plain
[root@aio ~(keystone_admin)]# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
26: vfw: <BROADCAST,MULTICAST> mtu 1450 qdisc noop state DOWN group default qlen 1000
    link/ether a2:e9:b4:b5:63:a0 brd ff:ff:ff:ff:ff:ff
27: vfw-in: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 32:0f:41:6a:6e:43 brd ff:ff:ff:ff:ff:ff
28: vfw-out: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3e:3c:59:c0:80:47 brd ff:ff:ff:ff:ff:ff
```

我们将原来VFW的IP:`192.168.100.6`配置在`br-int`上的`vfw`接口上，也就是Fake vRouter的单臂接口:
```bash
ip addr add 192.168.100.6/24 dev vfw
```

给`vfw-in`接口配置一个路由IP, 如: `100.100.100.100/24`:
```bash
ip addr add 100.100.100.100/24 dev vfw-in
```

给`vfw-out`接口配置同一个二层网络的路由IP, 如: `100.100.100.101/24`:
```bash
ip addr add 100.100.100.101/24 dev vfw-out
```

启动相应接口:
```bash
ip link set up dev vfw
ip link set up dev vfw-in
ip link set up dev vfw-out
```

关闭相应接口的`rp_filter`选项:
```bash
sysctl -w net.ipv4.conf.vfw.rp_filter=0
sysctl -w net.ipv4.conf.vfw-in.rp_filter=0
sysctl -w net.ipv4.conf.vfw-out.rp_filter=0
sysctl -w net.ipv4.conf.all.rp_filter=0
```

我们需要将`vfw`接口进入的数据包，由接口`vfw-in`转发至VFW虚拟机，数据包直接通过VFW虚拟机内部二层转发到`vfw-out`接口处，再由`vfw`接口回注到原来路径中。

首先添加一个路由表用于从`vfw`接口转发到`vfw-in`接口:
```bash
ip route add default via 100.100.100.101 dev vfw-in table 100
```

添加另一个路由表用于从`vfw-out`接口转发到`vfw`接口:
```bash
ip route add default via 192.168.100.1 dev vfw table 101
```

配置相应的路由规则:
```plain
[root@aio ~(keystone_admin)]# ip rule add iif vfw lookup 100
[root@aio ~(keystone_admin)]# ip rule add iif vfw-out lookup 101
[root@aio ~(keystone_admin)]# ip rule
0:    from all lookup local
32764:    from all iif vfw-out lookup 101
32765:    from all iif vfw lookup 100
32766:    from all lookup main
32767:    from all lookup default
```

至此时，我们所有的配置就都完成了。

我们从外向内访问, 访问成功:
```plain
[root@aio ~]# ping 172.24.4.4  -c 2
PING 172.24.4.4 (172.24.4.4) 56(84) bytes of data.
64 bytes from 172.24.4.4: icmp_seq=1 ttl=60 time=3.07 ms
64 bytes from 172.24.4.4: icmp_seq=2 ttl=60 time=2.03 ms

--- 172.24.4.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.033/2.555/3.077/0.522 ms
```

在VFW虚拟机上抓包看结果正常:
```plain
[root@vfw ~]# tcpdump -ieth0 -nn -e icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:15:49.837903 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 172.24.4.1 > 10.10.10.9: ICMP echo request, id 19718, seq 1, length 64
11:15:49.839749 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 172.24.4.1: ICMP echo reply, id 19718, seq 1, length 64
11:15:50.838751 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 172.24.4.1 > 10.10.10.9: ICMP echo request, id 19718, seq 2, length 64
11:15:50.840022 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 172.24.4.1: ICMP echo reply, id 19718, seq 2, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

再从app虚拟机访问外部，访问也正常:
```plain
$ ping 114.114.114.114 -c 2
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: seq=0 ttl=58 time=27.473 ms
64 bytes from 114.114.114.114: seq=1 ttl=82 time=27.177 ms

--- 114.114.114.114 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 27.177/27.325/27.473 ms
```

从VFW虚拟机上抓包结果正常:
```plain
[root@vfw ~]# tcpdump -ieth0 -nn -e icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:17:33.030527 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 39425, seq 0, length 64
11:17:33.055787 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 114.114.114.114 > 10.10.10.9: ICMP echo reply, id 39425, seq 0, length 64
11:17:34.031610 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 39425, seq 1, length 64
11:17:34.056536 32:0f:41:6a:6e:43 > 3e:3c:59:c0:80:47, ethertype IPv4 (0x0800), length 98: 114.114.114.114 > 10.10.10.9: ICMP echo reply, id 39425, seq 1, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

这种方法是比较Tricky的实现，只用于POC验证阶段。如果用于实际业务场景，需要修改`Neutron`代码或者独立实现AGENT完成相应操作。更为优雅的方式，则可以基于`networking-sfc`实现二层接入，后续再写文章来说明。
