---
layout: post
title: "OpenStack虚拟机实例关联多个浮动IP"
date: 2016-12-22 16:43:37 +0800
comments: true
categories: OpenStack
---
许多场景下我们的OpenStack虚拟机实例需要关联多个浮动IP。Horizon Dashboard不支持这样的操作，但是我们可以通过命令行和API完成。

可以以两种形式关联:

* 实例配置多个NIC，每个NIC关联一个浮动IP
* 实例只配置一个NIC，NIC上配置多个fixed IP, 每个浮动IP关联一个fixed IP

首先看第一种形式:

列出所有网络，找到实例所在网络和子网:
```plain
[root@localhost ~(keystone_admin)]# neutron net-list
+--------------------------------------+---------+------------------------------------------------------+
| id                                   | name    | subnets                                              |
+--------------------------------------+---------+------------------------------------------------------+
| 1ced55ea-d30f-4c1d-ba97-6a7bbbd9ecca | private | f805a609-da7c-4a79-98a4-1aefb6aca2a2 10.0.0.0/24     |
| 467b92e0-540d-4cac-86a6-999354c6b6c8 | 1       | 216fa519-973f-493f-a8ef-1b5f9e30335f 10.0.1.0/24     |
| 65bdc2e1-d3ba-4b20-912c-6f0a29aef5de | public  | 189e9999-4295-45a4-b6b2-438225249654 172.24.4.224/28 |
| 73569655-f522-4e3d-b2f3-f1a1fd83cc5e | 2       | c9015487-e913-4a31-bc14-ef3cb549c895 10.0.2.0/24     |
+--------------------------------------+---------+------------------------------------------------------+
```

<!--more-->

创建一个PORT, 并手动分配fixed IP
```plain
[root@localhost ~(keystone_admin)]# neutron port-create 467b92e0-540d-4cac-86a6-999354c6b6c8 --fixed-ip subnet_id=216fa519-973f-493f-a8ef-1b5f9e30335f,ip_address=10.0.1.9
Created a new port:
+-----------------------+---------------------------------------------------------------------------------+
| Field                 | Value                                                                           |
+-----------------------+---------------------------------------------------------------------------------+
| admin_state_up        | True                                                                            |
| allowed_address_pairs |                                                                                 |
| binding:host_id       |                                                                                 |
| binding:profile       | {}                                                                              |
| binding:vif_details   | {}                                                                              |
| binding:vif_type      | unbound                                                                         |
| binding:vnic_type     | normal                                                                          |
| created_at            | 2016-12-22T07:55:42Z                                                            |
| description           |                                                                                 |
| device_id             |                                                                                 |
| device_owner          |                                                                                 |
| extra_dhcp_opts       |                                                                                 |
| fixed_ips             | {"subnet_id": "216fa519-973f-493f-a8ef-1b5f9e30335f", "ip_address": "10.0.1.9"} |
| id                    | 19cf86b9-abad-4db9-9bc0-93de24c434af                                            |
| mac_address           | fa:16:3e:d4:13:01                                                               |
| name                  |                                                                                 |
| network_id            | 467b92e0-540d-4cac-86a6-999354c6b6c8                                            |
| project_id            | 54b6b28dcf454951be177b2efea86158                                                |
| revision_number       | 4                                                                               |
| security_groups       | 3c9d564e-9d2e-4729-beac-a5da10586d17                                            |
| status                | DOWN                                                                            |
| tenant_id             | 54b6b28dcf454951be177b2efea86158                                                |
| updated_at            | 2016-12-22T07:55:42Z                                                            |
+-----------------------+---------------------------------------------------------------------------------+
```
将创建的PORT添加到实例
```plain
[root@localhost ~(keystone_admin)]# nova interface-attach --port-id 19cf86b9-abad-4db9-9bc0-93de24c434af t1
```
查看实例的网络信息, 可以发现已经有两个IP地址:
```plain
[root@localhost ~(keystone_admin)]# nova show t1 |grep network
| 1 network                            | 10.0.1.7, 10.0.1.9                                       |
```
在实例控制台查看，确认两个网卡都存在
```plain
[centos@t1 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:2e:57:2d brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.7/24 brd 10.0.1.255 scope global dynamic eth0
       valid_lft 84015sec preferred_lft 84015sec
    inet6 fe80::f816:3eff:fe2e:572d/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether fa:16:3e:d4:13:01 brd ff:ff:ff:ff:ff:ff
```
手动给第二块网卡配置IP并启动:
```bash
[centos@t1 ~]$ sudo ip addr add 10.0.1.9/24 dev eth1
[centos@t1 ~]$ sudo ip link set up eth1
```
查看是否有可用的浮动IP
```plain
[root@localhost ~(keystone_admin)]# neutron floatingip-list
+--------------------------------------+------------------+---------------------+--------------------------------------+
| id                                   | fixed_ip_address | floating_ip_address | port_id                              |
+--------------------------------------+------------------+---------------------+--------------------------------------+
| 2d2e436e-9da0-473f-87ad-226b3d1562f0 | 10.0.2.20        | 172.24.4.236        | c5b34619-a05e-4f95-b5c6-8d23032bec2b |
| 9c9ca6c4-5759-42e2-9ed1-dfcf54364c69 | 10.0.2.9         | 172.24.4.231        | 26c81d33-faf6-4813-83a6-9e64342e2a82 |
| aaff81e3-3358-424c-ace0-7e8982d9487e | 10.0.2.21        | 172.24.4.237        | c5b34619-a05e-4f95-b5c6-8d23032bec2b |
| d016382a-5848-4978-944d-e9d1e7cda763 | 10.0.2.7         | 172.24.4.234        | d078ea06-5891-4957-9964-43b4f4ca55f2 |
| dfc8faa4-2860-4ec9-826a-b06b7d4f4f9d |                  | 172.24.4.230        |                                      |
+--------------------------------------+------------------+---------------------+--------------------------------------+
```
若没有则创建一个浮动IP
```plain
[root@localhost ~(keystone_admin)]# neutron floatingip-create public
Created a new floatingip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| created_at          | 2016-12-22T08:12:03Z                 |
| description         |                                      |
| fixed_ip_address    |                                      |
| floating_ip_address | 172.24.4.238                         |
| floating_network_id | 65bdc2e1-d3ba-4b20-912c-6f0a29aef5de |
| id                  | c90b989b-a24d-457f-9fd0-57781922f367 |
| port_id             |                                      |
| project_id          | 54b6b28dcf454951be177b2efea86158     |
| revision_number     | 1                                    |
| router_id           |                                      |
| status              | DOWN                                 |
| tenant_id           | 54b6b28dcf454951be177b2efea86158     |
| updated_at          | 2016-12-22T08:12:03Z                 |
+---------------------+--------------------------------------+
```
将浮动IP与Port进行关联:
```plain
[root@localhost ~(keystone_admin)]# neutron floatingip-associate c90b989b-a24d-457f-9fd0-57781922f367 19cf86b9-abad-4db9-9bc0-93de24c434af
Associated floating IP c90b989b-a24d-457f-9fd0-57781922f367
```

若是实例需要很多浮动IP，按上述方法需要创建许多PORT。这种情况下我们可以在PORT上配置多个fixed IP, 从而将多个浮动IP与同一Port上的多个fixed IP进行关联。

创建PORT时指定多个浮动IP:
```bash
neutron port-create 73569655-f522-4e3d-b2f3-f1a1fd83cc5e --fixed-ip subnet_id=c9015487-e913-4a31-bc14-ef3cb549c895,ip_address=10.0.2.20 --fixed-ip subnet_id=c9015487-e913-4a31-bc14-ef3cb549c895,ip_address=10.0.2.21
```
关联浮动IP时指明要关联的fixed IP:
```bash
neutron floatingip-associate --fixed-ip-address 10.0.2.11 dfc8faa4-2860-4ec9-826a-b06b7d4f4f9d 26c81d33-faf6-4813-83a6-9e64342e2a82
```
