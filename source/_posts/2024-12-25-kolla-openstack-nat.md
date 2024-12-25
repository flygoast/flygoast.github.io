---
title: kolla部署openstack环境中的异常NAT
date: 2024-12-25 19:48:24
tags:
- Network
- Docker
- OpenStack
categories: Network
---
最近发现`kolla`安装的`openstack(Ocata版本)`环境中，某虚拟网络上的虚拟机对外访问异常。经过调查，发现虚拟机外发数据包经过`安全组`的网桥后源地址被修改为宿主机的`IP`。

简化的网络拓扑如图:

{% img /images/2024-12-25/1.jpg %}

<!--more-->

在虚拟机的网络接口上抓包观察，数据包正常:

```text-plain
[root@control01 ~]# tcpdump -i tap812bd281-f5 -nn icmp

06:36:35.596833 IP 172.17.0.8 > 172.17.0.9: ICMP echo request, id 62465, seq 0, length 64
06:36:36.597025 IP 172.17.0.8 > 172.17.0.9: ICMP echo request, id 62465, seq 1, length 64
```

而在`安全组`的`veth`端抓包观察，数据包源IP已被修改为宿主机IP:

```text-plain
[root@control01 ~]# tcpdump -iqvb812bd281-f5 -nn icmp

06:36:35.596889 IP 10.10.0.10 > 172.17.0.9: ICMP echo request, id 62465, seq 0, length 64
06:36:36.597074 IP 10.10.0.10 > 172.17.0.9: ICMP echo request, id 62465, seq 1, length 64
```

`docker`的默认网段为`172.17.0.1/16`,  而该虚拟网络网段恰好和`docker`网段重叠, 因而怀疑是网络冲突造成。

```text-plain
[root@control01 ~]# ip addr show docker0
6: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
   link/ether 02:42:01:73:00:16 brd ff:ff:ff:ff:ff:ff
   inet 172.17.0.1/16 scope global docker0
      valid_lft forever preferred_lft forever
   inet6 fe80::42:1ff:fe73:16/64 scope link
      valid_lft forever preferred_lft forever
```

查看系统上的`iptables`规则，发现`docker`会创建一条`MASQUERADE`规则, 用于对`docker`容器访问外部IP的数据包进行`NAT`操作:

{% img /images/2024-12-25/2.jpg %}

但实际上，该条规则写的过于宽泛，认为只要源IP网段是`docker`接口所配置的网段，并且出口不是`docker`接口的数据都认为是`docker`容器内外出的数据包。

在`kolla`部署的`openstack`场景下，因为宿主机上`net.bridge.bridge-nf-call-iptables`参数是开启的，从虚拟机通过`安全组`网桥时也会进行`iptables`规则匹配，由于网段重叠，同样会命中该条规则，因而被错误地执行`MASQUERADE`操作。

该问题从根本上看，是由于`docker`的这条规则写的过于宽泛，应该更精确一些，明确针对来自于`docker`容器的数据包。因为在`POSTROUTING`链上不允许使用`-i`来指定来源设备，可以将条件拆成两个阶段。在`PREROUTING`阶段对来自于`docker`容器的数据包设置`mark`值，而在`POSTROUTING`阶段的`MASQUERADE`规则额外添加上`mark`值的匹配条件。

这需要修改`docker`的源码，具体修改本文不详述，这里简单手动修改规则进行验证:

```text-plain
iptables -t nat -I PREROUTING 1 -i docker0 -j MARK --set-mark 0x12345678
iptables -t nat -I POSTROUTING 3 -m mark --mark 0x12345678 -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
iptables -t nat -D POSTROUTING 4
```

添加后的`iptables`规则如图:

{% img /images/2024-12-25/3.jpg %}

这时再次从`安全组`网桥的`veth`端抓包观察, 可以看到数据包恢复正常:

```text-plain
[root@control01 ~]# tcpdump -iqvb812bd281-f5 -nn icmp

06:52:18.106610 IP 172.17.0.8 > 172.17.0.9: ICMP echo request, id 62977, seq 0, length 64
06:52:18.107116 IP 172.17.0.9 > 172.17.0.8: ICMP echo reply, id 62977, seq 0, length 64
06:52:19.106773 IP 172.17.0.8 > 172.17.0.9: ICMP echo request, id 62977, seq 1, length 64
06:52:19.107000 IP 172.17.0.9 > 172.17.0.8: ICMP echo reply, id 62977, seq 1, length 64
```

还有另一种解决方法。因为`openstack`所添加的`安全组`网桥，名称是有特征性的, 都以`qbr`为前缀，可以在`MASQUERADE`规则前，添加`iptables`规则跳过`安全组`网桥的流量:

```text-plain
iptables -t nat -I POSTROUTING 2 -o qbr+ -j RETURN
```

修改后的`iptables`规则如图:

{% img /images/2024-12-25/4.jpg %}

此时再次在`安全组`网桥的`veth`端抓包，确认数据包也恢复正常:

```text-plain
[root@control01 ~]# tcpdump -iqvb812bd281-f5 -nn icmp

07:01:07.755490 IP 172.17.0.8 > 172.17.0.9: ICMP echo request, id 63233, seq 0, length 64
07:01:07.756178 IP 172.17.0.9 > 172.17.0.8: ICMP echo reply, id 63233, seq 0, length 64
07:01:08.755598 IP 172.17.0.8 > 172.17.0.9: ICMP echo request, id 63233, seq 1, length 64
07:01:08.755821 IP 172.17.0.9 > 172.17.0.8: ICMP echo reply, id 63233, seq 1, length 64
```

很久没有研究过`openstack`，可能现在新版本的`openstack`已经不存在这种网桥形式的`安全组`实现了，因而也就不会存在这个问题。
