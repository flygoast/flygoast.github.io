---
layout: post
title: "RDO安装all-in-one模式OpenStack虚拟网络架构分析"
date: 2017-01-07 23:45:09 +0800
comments: true
categories: OpenStack
---
RDO是在RHEL，Fedora, CentOS等系统上部署OpenStack的方案。简单的试用或研究OpenStack可以选择all-in-one模式安装。这种模式将所有的OpenStack组件都部署在同一台机器上。本文来分析这种模式下虚拟网络的架构。

all-in-one模式安装完成时，RDO自动完成了以下工作:

* 创建了外部网络public: `172.24.4.224/28`
* 创建了租户demo
* 为租户demo创建了私有网络private: `10.0.0.0/24`
* 为租户demo创建路由器router1, 将private和public连接在一起

我们登录demo租户，创建两个虚拟机，网络拓朴图如下:

{% img /images/2017-01-07/1.png %}

本文基于该拓朴分析虚拟网络架构。

<!--more-->

首先导入租户demo的keystone RC文件:
```bash
source ./keystonerc_demo
```
查看虚拟机列表:
```bash
[root@localhost ~(keystone_demo)]# nova list
+--------------------------------------+------+--------+------------+-------------+------------------+
| ID                                   | Name | Status | Task State | Power State | Networks         |
+--------------------------------------+------+--------+------------+-------------+------------------+
| bd6cc17d-9128-49aa-8834-734e9cac0b80 | vm0  | ACTIVE | -          | Running     | private=10.0.0.5 |
| 1b61b896-3c10-4237-85c1-0ee46a79036b | vm1  | ACTIVE | -          | Running     | private=10.0.0.6 |
+--------------------------------------+------+--------+------------+-------------+------------------+
```
查看Port列表:
```bash
[root@localhost ~(keystone_demo)]# neutron port-list
+--------------------------------------+------+-------------------+---------------------------------------------------------------------------------+
| id                                   | name | mac_address       | fixed_ips                                                                       |
+--------------------------------------+------+-------------------+---------------------------------------------------------------------------------+
| 55e885f5-932f-4a24-bedd-c08fc6a3aeb3 |      | fa:16:3e:89:a4:48 | {"subnet_id": "f805a609-da7c-4a79-98a4-1aefb6aca2a2", "ip_address": "10.0.0.6"} |
| aa048a76-f7bf-4d92-ad0c-538ea54c4080 |      | fa:16:3e:ae:16:12 | {"subnet_id": "f805a609-da7c-4a79-98a4-1aefb6aca2a2", "ip_address": "10.0.0.5"} |
| af6b8f51-4650-4a13-b31c-5d12d2939d8e |      | fa:16:3e:f7:d8:ef | {"subnet_id": "f805a609-da7c-4a79-98a4-1aefb6aca2a2", "ip_address": "10.0.0.1"} |
| be7511c5-68f6-4a0d-a1b3-3c5ae225789a |      | fa:16:3e:e3:eb:66 | {"subnet_id": "f805a609-da7c-4a79-98a4-1aefb6aca2a2", "ip_address": "10.0.0.2"} |
+--------------------------------------+------+-------------------+---------------------------------------------------------------------------------+
```
VM0的Port为: `aa048a76-f7bf-4d92-ad0c-538ea54c4080`, VM1的Port为: `55e885f5-932f-4a24-bedd-c08fc6a3aeb3`

查看TAP设备:
```bash
[root@localhost ~(keystone_demo)]# ip link |grep tap
21: tapaa048a76-f7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master qbraa048a76-f7 state UNKNOWN mode DEFAULT qlen 500
25: tap55e885f5-93: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master qbr55e885f5-93 state UNKNOWN mode DEFAULT qlen 500
```
TAP设备根据Port的UUID前11位命名，VM0的虚拟网络接口为`tapaa048a76-f7`, VM1的虚拟网络接口为`tap55e885f5-93`。

接着，查看Linux bridge信息:
```bash
[root@localhost ~(keystone_demo)]# brctl show
bridge name    bridge id        STP enabled    interfaces
qbr55e885f5-93        8000.e68f141893fa    no        qvb55e885f5-93
                                                     tap55e885f5-93
qbraa048a76-f7        8000.d6c6d2e7f411    no        qvbaa048a76-f7
                                                     tapaa048a76-f7
```
接口`tapaa048a76-f7`挂接到了Linux bridge: `qbraa048a76-f7`上，而且bridge: `qbraa048a76-f7`还挂接了`qvbaa048a76-f7`。neutron中，Linux bridge设备以qbr命名，qvb和qvo表示分示veth pair设备的两端。qvb表示Linux bridge端，qvo表示OVS bridge端。

查看veth设备:
```bash
[root@localhost ~(keystone_demo)]# ip a |grep qvb
19: qvoaa048a76-f7@qvbaa048a76-f7: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master ovs-system state UP qlen 1000
20: qvbaa048a76-f7@qvoaa048a76-f7: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master qbraa048a76-f7 state UP qlen 1000
23: qvo55e885f5-93@qvb55e885f5-93: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master ovs-system state UP qlen 1000
24: qvb55e885f5-93@qvo55e885f5-93: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master qbr55e885f5-93 state UP qlen 1000
```
可以看到veth pair的对应关系，再来查看OVS bridge：
```bash
[root@localhost ~(keystone_demo)]# ovs-vsctl show
723d8ad6-f2dd-4955-a7b3-3b8ec602cb05
    Manager "ptcp:6640:127.0.0.1" 
        is_connected: true
    Bridge br-ex
        Port "qg-9395e4c9-54" 
            Interface "qg-9395e4c9-54" 
                type: internal
        Port br-ex
            Interface br-ex
                type: internal
    Bridge br-int
        Controller "tcp:127.0.0.1:6633" 
            is_connected: true
        fail_mode: secure
        Port "qvo55e885f5-93" 
            tag: 1
            Interface "qvo55e885f5-93" 
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qr-af6b8f51-46" 
            tag: 1
            Interface "qr-af6b8f51-46" 
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "tapbe7511c5-68" 
            tag: 1
            Interface "tapbe7511c5-68" 
                type: internal
        Port "qvoaa048a76-f7" 
            tag: 1
            Interface "qvoaa048a76-f7" 
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633" 
            is_connected: true
        fail_mode: secure
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
    ovs_version: "2.5.0" 
```
可以发现qvo开头的设备挂载到OVS bridge: `br-int`上。

之所以在TAP设备和OVS bridge之间添加一层Linux bridge，是因为OpenStack基于IPTABLES实现安全组，而OVS目前不支持在OVS bridge上直接挂接的TAP设备上配置IPTABLES规则。

从上述OVS bridge信息可以看到，挂接到`br-int`上的qvo设备配置了VLAN ID:1。neutron在`br-int`上基于VLAN来实现不同私有网络隔离。`br-int`上挂接的设备除了veth设备，还有设备`tapbe7511c5-68`和`qr-af6b8f51-46`。

其中TAP设备: `tapbe7511c5-68`为DHCP服务器使用的网络接口。neutron基于network namespace和dnsmasq来实现DHCP。

查看network namespace:
```bash
[root@localhost ~(keystone_demo)]# ip netns list
qrouter-a8f68832-ba0f-4176-9536-86fa6fd6ef1e
qdhcp-1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca
```
DHCP的namespace为`qdhcp-1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca`。

查看namespace中的设备:
```bash
[root@localhost ~(keystone_demo)]# ip netns exec qdhcp-1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
11: tapbe7511c5-68: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
    link/ether fa:16:3e:e3:eb:66 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global tapbe7511c5-68
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fee3:eb66/64 scope link
       valid_lft forever preferred_lft forever
```

再来查看dnsmasq进程:
```bash
[root@localhost ~(keystone_demo)]# ps aux |grep dnsmasq
nobody   13598  0.0  0.0  15556   940 ?        S    Jan02   0:00 dnsmasq --no-hosts --no-resolv --strict-order --except-interface=lo --pid-file=/var/lib/neutron/dhcp/1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca/host --addn-hosts=/var/lib/neutron/dhcp/1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca/addn_hosts --dhcp-optsfile=/var/lib/neutron/dhcp/1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca/opts --dhcp-leasefile=/var/lib/neutron/dhcp/1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca/leases --dhcp-match=set:ipxe,175 --bind-interfaces --interface=tapbe7511c5-68 --dhcp-range=set:tag0,10.0.0.0,static,86400s --dhcp-option-force=option:mtu,1450 --dhcp-lease-max=256 --conf-file= --domain=openstacklocal
root     32126  0.0  0.0 112652   976 pts/0    S+   09:11   0:00 grep --color=auto dnsmasq
```

可以确认dnsmasq进程基于接口`tapbe7511c5-68`发送和接收DHCP数据包。

`br-int`上还挂接着设备:`qr-af6b8f51-46`，qr设备属于qrouter的namespace: `qrouter-a8f68832-ba0f-4176-9536-86fa6fd6ef1e`
```bash
[root@localhost ~(keystone_demo)]# ip netns exec qrouter-a8f68832-ba0f-4176-9536-86fa6fd6ef1e ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
12: qr-af6b8f51-46: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
    link/ether fa:16:3e:f7:d8:ef brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 brd 10.0.0.255 scope global qr-af6b8f51-46
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fef7:d8ef/64 scope link
       valid_lft forever preferred_lft forever
13: qg-9395e4c9-54: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
    link/ether fa:16:3e:52:dd:06 brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.229/28 brd 172.24.4.239 scope global qg-9395e4c9-54
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe52:dd06/64 scope link
       valid_lft forever preferred_lft forever
```
虚拟机的默认网关为`10.0.0.1`, 配置在设备`qr-af6b8f51-46`上。虚拟机访问外网时，数据包会由该接口进入qrouter的namespace。

查看namespace内的路由信息:
```bash
[root@localhost ~(keystone_demo)]# ip netns exec qrouter-a8f68832-ba0f-4176-9536-86fa6fd6ef1e ip route
default via 172.24.4.225 dev qg-9395e4c9-54
10.0.0.0/24 dev qr-af6b8f51-46  proto kernel  scope link  src 10.0.0.1
172.24.4.224/28 dev qg-9395e4c9-54  proto kernel  scope link  src 172.24.4.229
```
可以看到qrouter的默认路由为从接口`qg-9395e4c9-54`发送至`172.24.4.225`。

从上边的OVS bridge信息看到: 接口`qg-9395e4c9-54`挂接在OVS bridge: `br-ex`上。OVS bridge默认存在一个同名的网络接口，来查看`br-ex`接口信息:
```bash
[root@localhost ~(keystone_demo)]# ip addr show br-ex
9: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 0a:c0:1c:ac:17:4f brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.225/28 scope global br-ex
       valid_lft forever preferred_lft forever
    inet6 fe80::8c0:1cff:feac:174f/64 scope link
       valid_lft forever preferred_lft forever
```
`br-ex`上配置了IP：`172.24.4.225`。可以看到，虚拟机访问外网的数据包会由qrouter发送到`br-ex`接口。

再来查看IPTABLES规则:
```bash
[root@localhost ~(keystone_demo)]# iptables -t nat -S POSTROUTING
-P POSTROUTING ACCEPT
-A POSTROUTING -j neutron-openvswi-POSTROUTING
-A POSTROUTING -j neutron-postrouting-bottom
-A POSTROUTING -j nova-api-POSTROUTING
-A POSTROUTING -s 172.24.4.224/28 -o eth0 -m comment --comment "000 nat" -j MASQUERADE
-A POSTROUTING -j nova-postrouting-bottom
```
可以发现IPTABLES会将源地址为网段`172.24.4.224/28`的数据包进行NAT由接口eth0发出。

经过上述分析，all-in-one模式的整体网络架构如图:

{% img /images/2017-01-07/2.png %}

私有网络内的两个VM之间的数据通信，直接由`br-int`转发。虚拟机访问外网时，数据包则通过qrouter发送到`br-ex`接口，再由IPTABLES完成NAT从物理网卡发出。

