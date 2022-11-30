---
title: NSX-T东西向路由
date: 2022-11-22 20:45:44
tags:
- NSX-T
categories: Virtualization
---
之前的文章[<<NSX-T路由逻辑介绍>>](/2020/07/13/nsx-t/)主介绍了`NSX-T`的路由逻辑, 举例介绍的是南北向网络路径, 介绍从`逻辑交换机/分段`到`Tire1逻辑路由器`, 再到`Tire0逻辑路由器`的过程.

本文来简要介绍一下两个`逻辑交换机`之间通过`Tire1逻辑路由器`通信的东西向路径.

实验拓扑如图:
{% img /images/2022-11-22/1.png %}

<!--more-->

虚拟机`t1`的`IP`为:`6.6.100.11`, `t2`的`IP`为:`6.6.200.12`.

`N-VDS`或者`DVS`上的端口以`Geneve`的`VNI`互相隔离, 因而一个`Geneve VNI`就决定了一个`逻辑交换机/分段`. 我的环境的两个逻辑交换机的`Geneve VNI`如图:
{% img /images/2022-11-22/2.png %}

可以看到`ls-geneve-100`的`VNI`为`65537`, `ls-geneve-200`的`VNI`为`65536`.

使用命令`net-vdl2 -l`查看`VNI`:
```bash
[root@esxi-01:~] net-vdl2 -l
Global States:
	Control Plane Out-Of-Sync:	No
	VXLAN UDP Port:	4789
	Geneve UDP Port:	6081
NSX VDS:	DSwitch
	VDS ID:	50 02 70 16 c2 cd 74 37-fb a6 ff 0b 1b cd 0e ee
	MTU:	1600
	Segment ID:	10.10.10.0
	Transport VLAN ID:	300
	VTEP Count:	1
	CDO status:	enabled (deactivated)
		VTEP Interface:	vmk10
			DVPort ID:	b58c174b-a07f-43a6-b0ca-7830de39f50f
			Switch Port ID:	67108877
			Endpoint ID:	0
			VLAN ID:	300
			Label:		10292
			Uplink Port ID:	2214592537
			Is Uplink Port LAG:	No
			IP:		10.10.10.101
			Netmask:	255.255.255.0
			Segment ID:	10.10.10.0
			GW IP:		10.10.10.1
			GW MAC:		ff:ff:ff:ff:ff:ff
			IP Acquire Timeout:	0
			Multicast Group Count:	0
			Is DRVTEP:	Yes
	Network Count:	3
		Logical Network:	65538
			Routing Domain:	00000000-0000-0000-0000-000000000000
			Multicast Routing Domain: 00000000-0000-0000-0000-000000000000
			Replication Mode:	Source Unicast
			Control Plane:	Enabled (Multicast Proxy,ARP proxy)
			Controller:	10.44.205.85 (up)
			MAC Entry Count:	0
			ARP Entry Count:	0
			Port Count:	1
		Logical Network:	65537
			Routing Domain:	98334210-1ec6-4176-a718-581908b718c5
			Multicast Routing Domain: 00000000-0000-0000-0000-000000000000
			Replication Mode:	MTEP Unicast
			Control Plane:	Enabled (Multicast Proxy,ARP proxy)
			Controller:	10.44.205.85 (up)
			MAC Entry Count:	0
			ARP Entry Count:	1
			Port Count:	2
		Logical Network:	65536
			Routing Domain:	98334210-1ec6-4176-a718-581908b718c5
			Multicast Routing Domain: 00000000-0000-0000-0000-000000000000
			Replication Mode:	MTEP Unicast
			Control Plane:	Enabled (Multicast Proxy,ARP proxy)
			Controller:	10.44.205.85 (up)
			MAC Entry Count:	0
			ARP Entry Count:	0
			Port Count:	2
	Routing Domain Count:	2
		Routing DomainID:	00000000-0000-0000-0000-000000000000
		Routing DomainID:	98334210-1ec6-4176-a718-581908b718c5
```

可以看到所有的`逻辑交换机`也都位于该虚拟交换机.

在`ESXi01`上查看`DVS`的端口信息:
```bash
[root@esxi-01:~] nsxdp-cli vswitch instance list
DvsPortset-1 (DSwitch)           50 02 70 16 c2 cd 74 37-fb a6 ff 0b 1b cd 0e ee
Total Ports:2560 Available:2540
  Client                         PortID          DVPortID                             MAC                  Uplink
  Management                     67108868                                             00:00:00:00:00:00    n/a
  vmnic0                         2214592520      10                                   00:00:00:00:00:00
  Shadow of vmnic0               67108873                                             00:50:56:5c:37:04    n/a
  vmk0                           67108876        1                                    00:50:56:b1:59:3e    vmnic0
  vmk10                          67108877        b58c174b-a07f-43a6-b0ca-7830de39f50f 00:50:56:69:15:41    vmnic1
  vmk50                          67108878        8b2a4724-274f-46d0-a99b-580352399aa9 00:50:56:61:3f:85    void
  vdr-vdrPort                    67108883        vdrPort                              02:50:56:56:44:52    vmnic1
  spf-spfPort                    67108886        spfPort50027016c2cd7437              02:50:56:56:45:52    vmnic1
  vmnic1                         2214592537      11                                   00:00:00:00:00:00
  Shadow of vmnic1               67108890                                             00:50:56:5f:1e:d7    n/a
  t1.eth0                        67108910        e25a8fa7-0c21-4dae-b252-6d22ef33c1c5 00:50:56:82:70:f0    vmnic1
  t3.eth0                        67108917        c932ef38-c49f-4e28-8672-6ca34db2b38c 00:50:56:82:a0:05    vmnic1
```

可以看到, 所有的`逻辑交换机`端口都接在同一个虚拟交换机上. `逻辑路由器(Logical Router)`由`SR: Service Router`和`DR: Distributed Router`构成。`DR`分布在相应`传输区域`的`传输节点`上，`SR`则部署在`Edge`节点中。上边交换机端口`vdrPort`是`ESXi`主机上`DR`实例接到虚拟交换机的端口, 它可以理解为是`trunk`端口. 所有`逻辑交换机`的广播域流量都可以从它通过.

需要注意的`vdrPort`的`MAC`地址在所有`传输节点`上都是相同的, 默认为`02:50:56:56:44:52`.

在`ESXi-01`主机上查看`DR`:
```bash
[root@esxi-01:~] nsxcli -c get logical-routers
Tue Nov 22 2022 UTC 03:53:42.083
                                  Logical Routers Summary
 ------------------------------------------------------------------------------------------
               VDR UUID                LIF num  Route num  Max Neighbors  Current Neighbors
 98334210-1ec6-4176-a718-581908b718c5     2         2          50000              3

```

接着查看`DR`的接口信息:
```bash
[root@esxi-01:~] nsxcli -c get logical-router 98334210-1ec6-4176-a718-581908b718c5 interfaces
Tue Nov 22 2022 UTC 03:57:19.784
                         Logical Router Interfaces
---------------------------------------------------------------------------
IPv6 DAD Status Legend:  [A: DAD_Sucess], [F: DAD_Duplicate], [T: DAD_Tentative], [U: DAD_Unavailable]

LIF UUID                 : 39c68523-a185-49a7-9f86-4792e6696a8f
Mode                     : [b'Routing']
Overlay VNI              : 65536
IP/Mask                  : 6.6.200.1/24
Mac                      : 02:50:56:56:44:52
Connected DVS            : DSwitch
Control plane enable     : True
Replication Mode         : 0.0.0.1
Multicast Routing        : [b'Enabled', b'Oper Down']
State                    : [b'Enabled']
Flags                    : 0x80388
DHCP relay               : Not enable
DAD-mode                 : ['LOOSE']
RA-mode                  : ['UNKNOWN']

LIF UUID                 : 4adea6ee-5dbf-4ff8-8fa4-6670bb70982f
Mode                     : [b'Routing']
Overlay VNI              : 65537
IP/Mask                  : 6.6.100.1/24
Mac                      : 02:50:56:56:44:52
Connected DVS            : DSwitch
Control plane enable     : True
Replication Mode         : 0.0.0.1
Multicast Routing        : [b'Enabled', b'Oper Down']
State                    : [b'Enabled']
Flags                    : 0x80388
DHCP relay               : Not enable
DAD-mode                 : ['LOOSE']
RA-mode                  : ['UNKNOWN']

```
可以看到`6.6.100.1`和`6.6.200.1`两个接口的`MAC`地址都为:`02:50:56:56:44:52`.

现在我们来看`6.6.100.11`到`6.6.200.12`的网络路径.

在`t1`上清空`ARP`信息, 然后`ping`虚拟机`t2`. 因为目标IP`6.6.200.12`不在相同子网内, 会先发送`ARP`请求来确认网关`6.6.100.1`的`MAC`地址.

我们在`t1.eth0`, `vdrPort`和`uplink`上进行抓包.

只有在`t1.eth0`端口上抓到`ARP`请求:
```bash
[root@esxi-01:~] pktcap-uw --switchport 67108910 --dir 2 -o - | tcpdump-uw -ner -
The switch port id is 0x0400002e.
pktcap: The output file is -.
pktcap: No server port specifed, select 7799 as the port.
pktcap: Local CID 2.
pktcap: Listen on port 7799.
reading from file -, link-type EN10MB (Ethernet)
pktcap: Accept...
pktcap: Vsock connection from port 1096 cid 2.
11:45:24.879494 00:50:56:82:70:f0 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 6.6.100.1 tell 6.6.100.11, length 46
11:45:24.879541 02:50:56:56:44:52 > 00:50:56:82:70:f0, ethertype ARP (0x0806), length 60: Reply 6.6.100.1 is-at 02:50:56:56:44:52, length 46
```

猜测虚拟交换机层面对虚拟子网网关实现了`ARP`代答, 这样发送向网关的流量导向本机的`vdrPort`, 尽管各个`ESXi`主机上的`vdrPort`的`MAC`地址都相同也不会冲突, 因为这样的`ARP`请求不会送到其他`ESXi`主机上.

接下来, 在虚拟机`t1`上长`ping` `t2`, 我们分别在`ESXi-01`和`ESXi-02`的`vdrPort`上抓包.

在发送方`t1`所在`ESXi-01`上的`vdrPort`, 可以看到两个`request`包, 但没有`reply`包:
```bash
12:17:25.381567 00:50:56:82:70:f0 > 02:50:56:56:44:52, ethertype IPv4 (0x0800), length 98: 6.6.100.11 > 6.6.200.12: ICMP echo request, id 9652, seq 11, length 64
12:17:25.381593 02:50:56:56:44:52 > 00:50:56:82:a6:ae, ethertype IPv4 (0x0800), length 98: 6.6.100.11 > 6.6.200.12: ICMP echo request, id 9652, seq 11, length 64
12:17:26.382613 00:50:56:82:70:f0 > 02:50:56:56:44:52, ethertype IPv4 (0x0800), length 98: 6.6.100.11 > 6.6.200.12: ICMP echo request, id 9652, seq 12, length 64
12:17:26.382645 02:50:56:56:44:52 > 00:50:56:82:a6:ae, ethertype IPv4 (0x0800), length 98: 6.6.100.11 > 6.6.200.12: ICMP echo request, id 9652, seq 12, length 64
```

而在虚拟机`t2`所在的`ESXi-02`上的`vdrPort`, 只有`reply`包:
```bash
12:17:25.588603 00:50:56:82:a6:ae > 02:50:56:56:44:52, ethertype IPv4 (0x0800), length 98: 6.6.200.12 > 6.6.100.11: ICMP echo reply, id 9652, seq 11, length 64
12:17:25.588627 02:50:56:56:44:52 > 00:50:56:82:70:f0, ethertype IPv4 (0x0800), length 98: 6.6.200.12 > 6.6.100.11: ICMP echo reply, id 9652, seq 11, length 64
12:17:26.590845 00:50:56:82:a6:ae > 02:50:56:56:44:52, ethertype IPv4 (0x0800), length 98: 6.6.200.12 > 6.6.100.11: ICMP echo reply, id 9652, seq 12, length 64
12:17:26.590873 02:50:56:56:44:52 > 00:50:56:82:70:f0, ethertype IPv4 (0x0800), length 98: 6.6.200.12 > 6.6.100.11: ICMP echo reply, id 9652, seq 12, length 64
```

因而数据包的路由是在数据包发送方主机上的`DR`实例来实现, 数据包到达目标主机后, 直接解封装送到目标虚拟机.

整体路径如图:
{% img /images/2022-11-22/3.png %}

所有`ESXi`主机上的`vdrPort`的`MAC`地址都一致, 且`vdrport`上可以接收到`uplink`所连接物理网络的数据包. 一般情况下该`MAC`地址并不会暴露到物理网络中, 但当虚拟交换机上的某`uplink`接口`down`掉, 启用`standby uplink`时, `ESXi`会广播发送`Reverse ARP`向物理交换机宣告这些`MAC`在该端口下, 这种情况下会导致`vdrPort`的`MAC`地址暴露到物理网络, 如:
```plain
14:53:52.919368 00:50:56:6c:e2:6a > ff:ff:ff:ff:ff:ff, ethertype Reverse ARP (0x8035), length 60: Reverse Request who-is 00:50:56:6c:e2:6a tell 00:50:56:6c:e2:6a, length 46
14:53:52.919379 02:50:56:56:44:52 > ff:ff:ff:ff:ff:ff, ethertype Reverse ARP (0x8035), length 60: Reverse Request who-is 02:50:56:56:44:52 tell 02:50:56:56:44:52, length 46
14:53:52.919397 00:50:56:6c:e2:6a > ff:ff:ff:ff:ff:ff, ethertype Reverse ARP (0x8035), length 60: Reverse Request who-is 00:50:56:6c:e2:6a tell 00:50:56:6c:e2:6a, length 46
14:53:52.919397 00:50:56:6c:e2:6a > ff:ff:ff:ff:ff:ff, ethertype Reverse ARP (0x8035), length 60: Reverse Request who-is 00:50:56:6c:e2:6a tell 00:50:56:6c:e2:6a, length 46
14:53:52.919406 00:50:56:53:71:23 > ff:ff:ff:ff:ff:ff, ethertype Reverse ARP (0x8035), length 60: Reverse Request who-is 00:50:56:53:71:23 tell 00:50:56:53:71:23, length 46
14:53:52.919409 2c:f0:5d:1d:b0:41 > ff:ff:ff:ff:ff:ff, ethertype Reverse ARP (0x8035), length 60: Reverse Request who-is 2c:f0:5d:1d:b0:41 tell 2c:f0:5d:1d:b0:41, length 46
```

当不同的`uplink`异常, 多台`ESXi`启用不同的`uplink`后, 该`MAC`会暴露在不同的物理交换机端口, 因而交换机可能会告警存在`mac-address flapping`.

参考:

* https://www.dclessons.com/virtual-physical-network-connectivity
* https://docs.vmware.com/en/VMware-NSX-T-Data-Center/3.2/migration/GUID-538774C2-DE66-4F24-B9B7-537CA2FA87E9.html
* https://itdeepdive.com/2020/11/nsx-t-packet-capture-east-west-traffic-on-overlay-segment/
* https://nielshagoort.com/2018/12/13/understanding-the-esxi-network-iochain/
* https://thevirtualunknown.co.uk/2017/02/14/vmware-nsx-iochain-and-how-packets-are-processed-within-the-kernel/
* https://y-network.jp/2020/12/13/nsx-t-073/
* http://notes.doodzzz.net/2017/07/17/vmware-nsx-and-3rd-party-integration-svm-failure-scenarios/
* https://vgandalf.com/2021/05/19/trace-this/
