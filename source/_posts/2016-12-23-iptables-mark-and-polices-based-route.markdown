---
layout: post
title: "基于IPTABLES MARK机制实现策略路由"
date: 2016-12-23 17:16:12 +0800
comments: true
tags:
- Iptables
- Mark
categories: Network
---
MARK是IPTABLES的一种规则目标，它用于给匹配了相应规则的数据包设置标签。然而，标签并不是设置于数据包内容中，而是设置在内核中数据包的载体上。如果需要在数据包内容中设置标签，可以使用TOS规则目标，它可以修改IP数据包头的TOS值。

MARK是一个32位整数值, MARK目标可以使用3种方法来设置mark值:

* `--set-mark value`: 直接设置mark值为value
* `--and-mark value`: 将mark值与value做位与运算后设置为新mark值
* `--or-mark value`: 将mark值与value做位或运算后设置为新mark值

比如，将从网络接口tun0进入的、目标端口为5222的TCP数据包设置mark值为1:
```bash
iptables -t mangle -A PREROUTING -j MARK --set-mark 1 -i tun0 -p tcp --dport 5222
```

<!--more-->

设置的mark值可用来设定策略路由。

比如，我们想把mark值为1的数据包交由网关`192.168.0.1`转发。

首先，确定一张空路由表，这里选定`300`:
```bash
[root@compute1 linux]# ip route show table 300
```
在表中添加路由条目:
```bash
[root@compute1 linux]# ip route add default via 192.168.0.1 table 300
```
查看，当前路由规则:
```bash
[root@compute1 linux]# ip rule list
0:         from all lookup local
32766:     from all lookup main
32767:     from all lookup default
```
为mark值为1的数据包指定路由表策略:
```bash
[root@compute1 linux]# ip rule add fwmark 0x1 table 300
```
再查看当前策略，确定已经生效:
```bash
[root@compute1 linux]# ip rule list
0:         from all lookup local
32765:     from all fwmark 0x1 lookup 300
32766:     from all lookup main
32767:     from all lookup default
```

通过这种方法，可以使用IPTABLES根据匹配规则设置mark, 再由路由模块根据mark值进行路由决策，从而实现复杂的策略路由。
