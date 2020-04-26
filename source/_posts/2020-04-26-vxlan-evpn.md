---
title: 基于BGP EVPN的VXLAN通信实践
date: 2020-04-26 09:30:43
tags: 
- VXLAN
categories: Network

---
VXLAN现在主要有以下几种类型的控制面:
* 基于集中的SDN控制器
* 基于IP多播实现洪泛与学习
* 基于单播的HER(Head-End Replication)洪泛与学习，即在入口VTEP处实现包复制，通过单播发送给需要洪泛的所有VTEP
* 基于BGP EVPN(Ethernet Virtual Private Network)

VMware NSX方案是基于SDN控制器。而之前的两篇关于VXLAN的文章<<[动态维护FDB表项实现VXLAN通信](/2020/04/20/vxlan-fdb/)>>和<<[VXLAN原理介绍与实例分析](/2017/05/21/vxlan/)>>中, 我们实验的方式是基于多播和单播的方案。本文我们来介绍基于BGP EVPN的VXLAN控制面。

BGP EVPN是一个标准的控制面协议，由[RFC7432](https://tools.ietf.org/html/rfc7432)定义。它支持三种数据面: MPLS、PBB、NVO(Network Virtualization Overlay)。VXLAN是NVO方案中的一种，它的EVPN应用由[RFC8365](https://tools.ietf.org/html/rfc8365)定义。BGP EVPN依赖[BGP协议](https://tools.ietf.org/html/rfc4271)和[MP-BGP扩展](https://tools.ietf.org/html/rfc4760)。BGP是支撑互联网的主要协议，用于在网络设备之间同步路由信息。MP-BGP扩展能够将BGP所同步信息扩展到多种协议，如IPv4, IPv6, L3VPN和EVPN等。EVPN地址族用来传递MAC地址以及这些MAC地址所挂接的设备IP。

<!--more-->

EVPN方案相较其他的控制面方案具备这些优势:
* 没有集中的SDN控制器，部署简单
* 扩展性、稳定性、不同厂商设备间的互操作性都更高
* 可以提供细粒度的流量控制策略

基于EVPN的VXLAN方案中，VTEP通常做为第一跳路由配置在ToR交换机(虚拟化场景中可以是Hypervisor上的虚拟交换机)。当一个本地交换机在某VLAN学习到本地MAC地址（通过Gratuitous ARP或首个数据包)后, 将MAC地址放到交换机的FDB表中。每个Local VLAN会映射到一个VNI。本地的MP-BGP进程收集到本地VTEP所配置的VNI信息、每个VNI下的MAC地址，然后将这些信息通过MP-BGP协议宣告给其他的VTEP。其中:
* VTEP信息通过Type 3路由宣告
* 每个VNI下的本地MAC地址通过Type 2路由宣告

在接收端, BGP进程将接收到的MAC地址存放在BGP表中。如果接收到的路由中的RT community与本地VNI的导入RT相匹配，这些MAC地址就会被放入本地交换的FDB中。通过这个流程，其他的VTEP就具备了本地VTEP上的MAC地址信息, 从而构造完成VXLAN的FDB表项，后续数据面转发则可以直接从MAC地址、VNI找到相应的VTEP。
表项格式可简化表达为:
```plain
<MAC> <VNI> <REMOTE VTEP>
```

为了在所有VTEP之间进行路由同步，需要采用FullMesh方式，即任意两个VTEP之间都需要建立BGP会话。在大规模VXLAN环境中这样配置非常复杂，为了降低配置复杂度，通常会引入RR(Router Reflector)。这样，每个VTEP都只需要与RR建立BGP会话，RR会将从某个BGP Peer所接收到的路由信息转发给其他的所有BGP Peer。

开源项目[FRRoutring](https://frrouting.org/)是一个功能丰富的Linux路由软件，它实现了EVPN协议。我们在Linux上基于FRR来实验基于BGP EVPN的VXLAN通信过程。实验结构与之前的文章一致，只是在两台主机上添加了FRR软件和一台RouterReflector虚拟机，如下图:
{% img /images/2020-04-26/1.png %}


在实际EVPN的部署中，可以使用：
* eBGP: 不同的VTEP的AS号不同
* iBGP: VTEP的AS号相同

我们这里的示例使用iBGP, VTEP AS号都为: 65000。

首先我们在虚拟机`192.168.33.12`上使用FRR作为Router Reflector。
在FRR的BGP配置文件中添加如下配置:
```plain
router bgp 65000
  bgp router-id 192.168.33.12
  bgp cluster-id 192.168.33.12
  bgp log-neighbor-changes
  no bgp default ipv4-unicast
  neighbor fabric peer-group
  neighbor fabric remote-as 65000
  neighbor fabric capability extented-nexthop
  neighbor fabric update-source 192.168.33.12
  bgp listen range 192.168.33.0/24 peer-group fabric
  !
  address-family l2vpn evpn
    neighbor fabric activate
    neighbor fabric route-reflector-client
  exit-address-family
  !
  exit
!
```
我们定义一个peer-group: `fabric`, 并且使用FRR的动态peer特性接收来自所有`192.168.33.0/24`网段且ASN为`65000`的BGP peer连接。

接下来我们在两台虚拟机上配置VTEP。我们需要在VXLAN接口上禁用源地址学习, 只依赖EVPN来同步FDB。在Host1上使用脚本创建本地交换机和VXLAN设备:
```bash
sysctl -w net.ipv4.ip_forward=1
ip netns add ns1
ip link add veth1 type veth peer name eth0 netns ns1
ip netns exec ns1 ip link set eth0 up
ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip addr add 3.3.3.3/24 dev eth0
ip link set up dev veth1
ip link add br1 type bridge
ip link set br1 up
ip link set veth1 master br1
ip link add vxlan100 type vxlan id 100 dstport 4789 local 192.168.33.15 nolearning
ip link set vxlan100 master br1
ip link set up vxlan100
```

在Host2上同样配置:
```bash
sysctl -w net.ipv4.ip_forward=1
ip netns add ns1
ip link add veth1 type veth peer name eth0 netns ns1
ip netns exec ns1 ip link set eth0 up
ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip addr add 3.3.3.4/24 dev eth0
ip link set up dev veth1
ip link add br1 type bridge
ip link set br1 up
ip link set veth1 master br1
ip link add vxlan100 type vxlan id 100 dstport 4789 local 192.168.33.16 nolearning
ip link set vxlan100 master br1
ip link set up vxlan100
```

此时我们从`192.168.33.15`机器上的`ns1`来访问`192.168.33.16`机器中的`ns1`中的IP, 访问不成功:
```plain
root@ubuntu-focal:/home/vagrant/workspace# ip netns exec ns1 ping -c2 3.3.3.4
PING 3.3.3.4 (3.3.3.4) 56(84) bytes of data.
From 3.3.3.3 icmp_seq=1 Destination Host Unreachable
From 3.3.3.3 icmp_seq=2 Destination Host Unreachable

--- 3.3.3.4 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1028ms
pipe 2
```

然后我们确认两台主机上的需要测试的虚拟网卡的MAC地址:
```plain
root@ubuntu-focal:/home/vagrant# ip netns exec ns1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 7a:bb:b4:6a:55:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

```plain
root@ubuntu-bionic:/home/vagrant# ip netns exec ns1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether e2:38:ad:ed:8f:9e brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

接下来我们配置Host1上的FRR配置, 添加:
```plain
log syslog informational
!
line vty
!
router bgp 65000
  bgp router-id 192.168.33.15
  no bgp default ipv4-unicast
  neighbor fabric peer-group
  neighbor fabric remote-as 65000
  neighbor fabric capability extented-nexthop
  ! BGP sessions with route reflectors
  neighbor 192.168.33.12 peer-group fabric
  !
  address-family l2vpn evpn
    neighbor fabric activate
    advertise-all-vni
  exit-address-family
  !
  exit
!
```

然后启动FRR:
```bash
systemctl start frr
```

在VTEP上我们建立与RR: `192.168.33.12`的BGP会话，配置`advertise-all-vni`指令去向其他peer宣告所有的本地VNI。

在Host2上也作同样的配置。

此时再次在Host1上执行ping, 访问成功:
```plain
root@ubuntu-focal:/home/vagrant/workspace# ip netns exec ns1 ping -c2 3.3.3.4
PING 3.3.3.4 (3.3.3.4) 56(84) bytes of data.
64 bytes from 3.3.3.4: icmp_seq=1 ttl=64 time=0.677 ms
64 bytes from 3.3.3.4: icmp_seq=2 ttl=64 time=0.731 ms

--- 3.3.3.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1007ms
rtt min/avg/max/mdev = 0.677/0.704/0.731/0.027 ms
```

此时, 我们在RR中通过`vtysh`查看BGP信息:
```plain
centos3# show bgp summary

L2VPN EVPN Summary:
BGP router identifier 192.168.33.12, local AS number 65000 vrf-id 0
BGP table version 0
RIB entries 3, using 552 bytes of memory
Peers 2, using 41 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
*192.168.33.15  4      65000       7       9        0    0    0 00:03:00            2
*192.168.33.16  4      65000       7       9        0    0    0 00:02:42            2

Total number of neighbors 2
* - dynamic neighbor
2 dynamic neighbor(s), limit 100
```

可以看到两台主机都建立了到RR的BGP会话。

接着查看evpn路由信息:
```plain
centos3# show  bgp evpn route
BGP table version is 10, local router ID is 192.168.33.12
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
                    Extended Community
Route Distinguisher: 192.168.33.15:2
*>i[2]:[0]:[48]:[7a:bb:b4:6a:55:02]
                    192.168.33.15                 100      0 i
                    RT:65000:100 ET:8
*>i[3]:[0]:[32]:[192.168.33.15]
                    192.168.33.15                 100      0 i
                    RT:65000:100 ET:8
Route Distinguisher: 192.168.33.16:2
*>i[2]:[0]:[48]:[e2:38:ad:ed:8f:9e]
                    192.168.33.16                 100      0 i
                    RT:65000:100 ET:8
*>i[3]:[0]:[32]:[192.168.33.16]
                    192.168.33.16                 100      0 i
                    RT:65000:100 ET:8

Displayed 4 prefixes (4 paths)
```

可以看到各接收到来自Host1和Host2的两条路由信息，分别是Type 2和Type 3。

在Host1上通过`vtysh`查看vxlan接口:
```plain
ubuntu-focal# show interface vxlan100
Interface vxlan100 is up, line protocol is up
  Link ups:       0    last: (never)
  Link downs:     0    last: (never)
  vrf: default
  index 6 metric 0 mtu 1500 speed 0
  flags: <UP,BROADCAST,RUNNING,MULTICAST>
  Type: Ethernet
  HWaddr: ae:46:52:bd:92:2e
  inet6 fe80::ac46:52ff:febd:922e/64
  Interface Type Vxlan
  VxLAN Id 100 VTEP IP: 192.168.33.15 Access VLAN Id 1

  Master interface: br1
```
可以看到VTEP信息，VNI为`100`，所属的bridge为`br1`

查看VNI:100下的本地MAC地址:
```plain
ubuntu-focal# show evpn mac vni 100
Number of MACs (local and remote) known for this VNI: 3
MAC               Type   Intf/Remote VTEP      VLAN  Seq #'s
7a:bb:b4:6a:55:02 local  veth1                       0/0
e2:38:ad:ed:8f:9e remote 192.168.33.16               0/0
06:a7:49:0e:8b:48 local  br1                   1     0/0
```

可以看到本地MAC`7a:bb:b4:6a:55:02`被发现，远端的MAC地址`e2:38:ad:ed:8f:9e`也被发现。

在Host1上查看EVPN路由信息，可以看到接收到了来自Host2所宣告的两条路由信息:
```plain
ubuntu-focal# show bgp evpn route
BGP table version is 6, local router ID is 192.168.33.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
                    Extended Community
Route Distinguisher: 192.168.33.15:2
*> [2]:[0]:[48]:[7a:bb:b4:6a:55:02]
                    192.168.33.15                      32768 i
                    ET:8 RT:65000:100
*> [3]:[0]:[32]:[192.168.33.15]
                    192.168.33.15                      32768 i
                    ET:8 RT:65000:100
Route Distinguisher: 192.168.33.16:2
*>i[2]:[0]:[48]:[e2:38:ad:ed:8f:9e]
                    192.168.33.16            0    100      0 i
                    RT:65000:100 ET:8
*>i[3]:[0]:[32]:[192.168.33.16]
                    192.168.33.16            0    100      0 i
                    RT:65000:100 ET:8

Displayed 4 prefixes (4 paths)
```

在Host1上查看VXLAN接口`vxlan100`的FDB:
```plain
root@ubuntu-focal:/home/vagrant# bridge fdb show dev vxlan100
e2:38:ad:ed:8f:9e vlan 1 extern_learn master br1
e2:38:ad:ed:8f:9e extern_learn master br1
ae:46:52:bd:92:2e vlan 1 master br1 permanent
ae:46:52:bd:92:2e master br1 permanent
00:00:00:00:00:00 dst 192.168.33.16 self permanent
e2:38:ad:ed:8f:9e dst 192.168.33.16 self extern_learn
```

可以看到EVPN接收的MAC地址`e2:38:ad:ed:8f:9e`已经写入到FDB中。

在Host2上也可以发现Host1上的MAC记录及相关路由信息。

参考链接:
* https://umairhoodbhoy.net/2014/10/29/head-end-replication-and-vxlan-compliance/
* https://www.sdnlab.com/19717.html
* https://vincent.bernat.ch/en/blog/2017-vxlan-bgp-evpn
* https://vincent.bernat.ch/en/blog/2017-vxlan-linux
