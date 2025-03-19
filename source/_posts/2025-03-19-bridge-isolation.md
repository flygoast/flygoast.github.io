---
title: 相同网桥上的网络隔离
date: 2025-03-19 18:20:58
tags:
- Network
- Docker
- Virtualization
categories: Network
---
我们的`oVirt`虚拟化平台上有一个需求，需要对同一网桥上的虚拟机之间进行网络隔离。

参考`Docker`实现中对于不同网桥的网络隔离，可以简单的采用`iptables`规则来实现。

`Docker`在`iptables`的`filter`表的`FORWARD`链的规则如下:

```plain
[root@localhost ~]# iptables -nL -v
Chain INPUT (policy ACCEPT 90 packets, 6017 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 109 packets, 6560 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

<!--more-->

当网桥流量流经`FORWARD`链时，会流经到`DOCKER-ISOLATION-STAGE-1`，该链主要用于对不同网络进行过滤。通过其中的规则:
```plain
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
```
来自`docker0`网桥但目的不是`docker0`网桥的流量会流经`DOCKER-ISOLATION-STAGE-2`。

`DOCKER-ISOLATION-STAGE-2`中的默认规则为:
```plain
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
```
它表示将来其他网络并目的网络为`docker0`的数据包丢弃。

如果在`Docker`环境中，希望实现开头提到的相同网桥的网络隔离，则可以直接在`DOCKER-ISOLATION-STAGE-1`中添加规则:
```plain
iptables -I DOCKER-ISOLATION-STAGE-1 -i docker0 -o docker0 -j DROP
```

这样就可以实现`docker0`上的容器之间的网络隔离。

因为网桥是二层网络设备而`iptables`是三层网络过滤机制，需要依赖内核模块`br_netfilter`的参数:
```plain
net.bridge.bridge-nf-call-iptables
```
开启网桥流量的`iptables`过滤。

`Docker`本身依赖这个机制，启动服务时会自动开启。在`oVirt`环境上我们需要手动开启:
```plain
modprobe br_netfilter
```
确认`bridge-nf-call-iptables`参数为`1`:
```plain
[root@localhost ~]# sysctl -a |grep bridge-nf-call
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

但在`oVirt`环境中需要注意的是，最好将规则添加到`FORWARD`链的最前边。这是因为`oVirt`中的`FORWARD`链中的规则使用了`-g`:
```plain
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i lo -j ACCEPT
-A FORWARD -j FORWARD_direct
-A FORWARD -j FORWARD_IN_ZONES_SOURCE
-A FORWARD -j FORWARD_IN_ZONES
-A FORWARD -j FORWARD_OUT_ZONES_SOURCE
-A FORWARD -j FORWARD_OUT_ZONES
-A FORWARD -m conntrack --ctstate INVALID -j DROP
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -i ovirtmgmt -o ovritmgmt -j DROP
-A OUTPUT -o lo -j ACCEPT
-A OUTPUT -j OUTPUT_direct
-A FORWARD_IN_ZONES -i ovirtmgmt -g FWDI_public
-A FORWARD_IN_ZONES -g FWDI_public
-A FORWARD_OUT_ZONES -o ovirtmgmt -g FWDO_public
-A FORWARD_OUT_ZONES -g FWDO_public
-A FWDI_public -j FWDI_public_log
-A FWDI_public -j FWDI_public_deny
-A FWDI_public -j FWDI_public_allow
-A FWDI_public -p icmp -j ACCEPT
-A FWDO_public -j FWDO_public_log
-A FWDO_public -j FWDO_public_deny
-A FWDO_public -j FWDO_public_allow
-A INPUT_ZONES -i ovirtmgmt -g IN_public
-A INPUT_ZONES -g IN_public
```

`-g`和`-j`作用类似，都是在规则匹配成功后进行跳转，区别在于:

* `-j`跳转之后，如果在目标链里没有匹配到任何规则，就会回到原来链的下一条规则继续匹配
* `-g`跳转之后，如果在目标链里没有匹配到任何规则，不会回到原来链的下一条规则，而是直接结束当前规则链的匹配过程。

如果将隔离规则添加到`FORWARD`链最后，则可能得不到匹配的机会。

因而，假设`oVirt`上需要隔离的网桥名称为`wan0`, 则需要在`filter`表的`FORWARD`链中添加规则:
```plain
iptables -I FORWARD -i wan0 -o wan0 -j DROP
```
就能实现相同网桥上的网络隔离。

这里我们测试使用`iptables`命令进行规则添加，而在程序中为了持久化写入规则，则可以使用`firewall-cmd`进行规则添加。
