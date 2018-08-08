---
layout: post
title: "OpenStack私有网络旁路部署虚拟防火墙"
date: 2018-08-08 10:10:18 +0800
comments: true
categories: OpenStack
---
之前的文章<<[OpenStack私有网络安全防护](/blog/2017/08/19/traffic-broker/)>>介绍了在私有网络中将虚拟防火墙(以下称为`VFW`)串联接入的方法。本文来介绍旁路部署`VFW`, 通过策略路由实现防火墙接入的方法。

逻辑网络拓扑如下:

{% img /images/2018-08-08/1.png %}

<!--more-->

我们在OpenStack环境中创建一个CentOS7的实例来模拟`VFW`, 部署完网络拓扑如下:

{% img /images/2018-08-08/2.png %}

我们需要将CentOS7的路由转发功能开启:
```bash
sysctl -w net.ipv4.ip_forward=1
```

此外，OpenStack网络上有`Anti Mac-Spoofing`的机制，默认情况下，只有数据包的`MAC`、`IP`与虚拟机网络接口的`MAC`、`IP`相匹配时才允许通过。因而我们需要配置`VFW`网络接口的`可用地址对`。将其IP设置为`0.0.0.0/0`, `MAC`为接口本身`MAC`，这样将允许所有数据包通过，如下图所示:

{% img /images/2018-08-08/3.png %}

为了将`VFW`串入网络路径中，我们需要在拓扑图中私有网络`demo`的`vRouter`上配置策略路由。OpenStack不支持配置策略路由, 我们只能手动在`vRouter`的`net namespace`中配置。

我们登录进`vRouter`的`namespace`:
```bash
ip netns exec qrouter-b2f289b6-5e7b-4f56-bf5d-ac01629eb333 bash
```

查看网络接口:
```plain
[root@aio ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
11: qr-a74cdf71-65: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:26:80:22 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global qr-a74cdf71-65
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe26:8022/64 scope link
       valid_lft forever preferred_lft forever
13: qg-c0eea9fd-62: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:d1:1f:45 brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.5/24 brd 172.24.4.255 scope global qg-c0eea9fd-62
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fed1:1f45/64 scope link
       valid_lft forever preferred_lft forever
21: qr-16448d15-00: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:84:42:72 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global qr-16448d15-00
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe84:4272/64 scope link
       valid_lft forever preferred_lft forever
```

添加一个专用路由表，将默认路由指向`VFW`所连接的`vRouter`接口:
```plain
[root@aio ~]# ip route add default via 192.168.100.6 dev qr-16448d15-00 table 100
[root@aio ~]# ip route show table 100
default via 192.168.100.6 dev qr-16448d15-00
```

添加入方向策略路由:
```bash
ip rule add iif qg-c0eea9fd-62 to 10.10.10.0/24 lookup 100
```

添加出方向策略路由:
```bash
ip rule add iif qr-a74cdf71-65 lookup 100
```

此时路由规则如下:
```plain
[root@aio ~]# ip rule
0:    from all lookup local
32764:    from all iif qr-a74cdf71-65 lookup 100
32765:    from all to 10.10.10.0/24 iif qg-c0eea9fd-62 lookup 100
32766:    from all lookup main
32767:    from all lookup default
```

我们给虚拟机实例绑定一个浮动IP:`172.24.4.4`, 然后从外部访问浮动IP，访问成功:
```plain
[root@aio ~]# ping 172.24.4.4 -c 2
PING 172.24.4.4 (172.24.4.4) 56(84) bytes of data.
64 bytes from 172.24.4.4: icmp_seq=1 ttl=61 time=9.94 ms
64 bytes from 172.24.4.4: icmp_seq=2 ttl=61 time=3.72 ms

--- 172.24.4.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 3.725/6.837/9.949/3.112 ms
```

在`VFW`上的抓包结果也符合预期:
```plain
[root@vfw ~]# tcpdump -ieth0 -nn icmp -e
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

12:52:22.138927 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 172.24.4.1 > 10.10.10.9: ICMP echo request, id 18352, seq 1, length 64
12:52:22.139199 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 172.24.4.1 > 10.10.10.9: ICMP echo request, id 18352, seq 1, length 64
12:52:22.142383 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 172.24.4.1: ICMP echo reply, id 18352, seq 1, length 64
12:52:22.143685 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 172.24.4.1: ICMP echo reply, id 18352, seq 1, length 64
12:52:23.138975 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 172.24.4.1 > 10.10.10.9: ICMP echo request, id 18352, seq 2, length 64
12:52:23.139423 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 172.24.4.1 > 10.10.10.9: ICMP echo request, id 18352, seq 2, length 64
12:52:23.141067 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 172.24.4.1: ICMP echo reply, id 18352, seq 2, length 64
12:52:23.141665 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 172.24.4.1: ICMP echo reply, id 18352, seq 2, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
```

我们再从业务虚拟机向外访问, 访问却是失败的:
```plain
$ ping -c 2 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes

--- 114.114.114.114 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

我的实验环境为All in One环境，在`br-ex`上抓包发现, 数据包的源IP没有被SNAT策略替换为正确的IP:
```plain
[root@aio ~]# tcpdump -i br-ex -nn -e icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br-ex, link-type EN10MB (Ethernet), capture size 262144 bytes

08:57:43.965997 fa:16:3e:d1:1f:45 > f2:3e:7c:35:ae:4e, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 33537, seq 0, length 64
08:57:44.965966 fa:16:3e:d1:1f:45 > f2:3e:7c:35:ae:4e, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 33537, seq 1, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```

这是由于OpenStack的虚拟网络实现上使用`IPTABLES`规则并依赖`CONNTRACK`机制实现`NAT`。SNAT实现上只在`CONNTRACK`状态为`NEW`时才会生成NAT所需的状态信息，后续被跳踪的该连接的数据包直接使用生成的NAT状态信息。我们从业务虚拟机向外访问，数据包到达`vRouter`上的接口时，`CONNTRACK`模块生成`CONNTRACK`表项，而此时出口接口为`VFW`路由接口，并不需要做`SNAT`，因而不会生成NAT状态信息。而从`VFW`路由接口再次到达vrouter时，`CONNTRACK`表项已经生成，不再处理NAT信息，而是直接将数据包发送出去。出入路由外网接口的数据包依赖`CONNTRACK`机制完成`NAT`，因而我们不能完全禁用`CONNTRACK`, 我们只能在业务子网接口和`VFW`子网接口之间的数据包取消`CONNTRACK`。

在`vRouter`中添加如下两条`iptables`规则来禁用`conntrack`:
```bash
iptables -t raw -A PREROUTING -i qr-a74cdf71-65 -j NOTRACK
iptables -t raw -A PREROUTING -i qr-16448d15-00 -d 10.10.10.0/24 -j NOTRACK
```

此时，我们再来测试, 访问成功:
```plain
$ ping -c 2 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: seq=0 ttl=53 time=30.849 ms
64 bytes from 114.114.114.114: seq=1 ttl=73 time=29.529 ms

--- 114.114.114.114 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 29.529/30.189/30.849 ms
```

从`VFW`上的抓包结果可以看出符合预期:
```plain
[root@vfw ~]# tcpdump -ieth0 -nn  icmp -e
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

09:00:07.099242 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 23809, seq 0, length 64
09:00:07.099515 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 23809, seq 0, length 64
09:00:07.127682 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 114.114.114.114 > 10.10.10.9: ICMP echo reply, id 23809, seq 0, length 64
09:00:07.128011 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 114.114.114.114 > 10.10.10.9: ICMP echo reply, id 23809, seq 0, length 64
09:00:08.100084 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 23809, seq 1, length 64
09:00:08.100456 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 10.10.10.9 > 114.114.114.114: ICMP echo request, id 23809, seq 1, length 64
09:00:08.127817 fa:16:3e:84:42:72 > fa:16:3e:73:c1:de, ethertype IPv4 (0x0800), length 98: 114.114.114.114 > 10.10.10.9: ICMP echo reply, id 23809, seq 1, length 64
09:00:08.128046 fa:16:3e:73:c1:de > fa:16:3e:84:42:72, ethertype IPv4 (0x0800), length 98: 114.114.114.114 > 10.10.10.9: ICMP echo reply, id 23809, seq 1, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
```

本文中的`VFW`接入方式为3层转发模式，而有些厂商的`VFW`只支持二层透明接入，这种情况下如何在OpenStack环境中应用呢？我后续再来写文章来说明。
