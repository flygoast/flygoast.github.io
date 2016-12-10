---
layout: post
title: "Linux路由分析"
date: 2016-12-10 15:24:34 +0800
comments: true
categories: OpenStack
---
Linux默认情况下会丢掉不属于本机IP的数据包。将`net.ipv4.ip_forward`设置为1后，会开启路由功能，Linux会像路由器一样对不属于本机的IP数据包进行路由转发。

路由的基本流程为: 收到数据包之后，解析出目的IP，判断是否是本机IP。如果是本机IP，则交由上层传输层处理。如果不是本机IP，则通过查找路由表找到合适的网络接口将IP数据包转发出去。

Linux上通过路由规则和路由表配合来实现路由流程, 处理逻辑如下:

* 按路由规则优先级, 根据规则匹配条件找到需要匹配的路由表
* 根据路由表中条目进行匹配的结果进行转发
* 若路由表中没有匹配到满足的路由条目，则处理下一路由规则

<!--more-->

路由规则由三部分构成:

* Priority: Linux上可以添加多条路由规则，根据优先级数字从小到大依次进行匹配，从0到32767
* Selector: 对IP数据包进行匹配的条件，如”from all” 表示所有IP数据包
* Action: 对IP数据包执行的动作，如”lookup local”表示需要查找路由表local进行处理

Linux支持多个路由表，其中有三个默认生成的路由表:

* local: 处理本地IP和广播地址路由，local路由表只由kernel维护，不能更改和删除
* main: 处理所有非策略路由，不指定路由表名时默认使用的路由表
* default: 所有其他路由表都没有匹配到的情况下，根据该表中的条目进行处理

首先，我们来查看路由规则:
```bash
[root@compute1 ~]# ip rule list
0:        from all lookup local
32766:    from all lookup main
32767:    from all lookup default
```
可以看到默认情况下存在3条路由规则，分别对应上述的三张路由表。路由选路时，首先处理规则0，对所有的IP数据包查找路由表local。我们来看local路由表的内容:
```bash
[root@compute1 ~]# ip route show table local
broadcast 127.0.0.0 dev lo  proto kernel  scope link  src 127.0.0.1
local 127.0.0.0/8 dev lo  proto kernel  scope host  src 127.0.0.1
local 127.0.0.1 dev lo  proto kernel  scope host  src 127.0.0.1
broadcast 127.255.255.255 dev lo  proto kernel  scope link  src 127.0.0.1
broadcast 172.16.88.0 dev eno33554976  proto kernel  scope link  src 172.16.88.130
local 172.16.88.130 dev eno33554976  proto kernel  scope host  src 172.16.88.130
broadcast 172.16.88.255 dev eno33554976  proto kernel  scope link  src 172.16.88.130
broadcast 172.16.96.0 dev eno16777736  proto kernel  scope link  src 172.16.96.131
local 172.16.96.131 dev eno16777736  proto kernel  scope host  src 172.16.96.131
broadcast 172.16.96.255 dev eno16777736  proto kernel  scope link  src 172.16.96.131
```
local表示本机IP，broadcast表示广播地址。local表由kernel来维护，用户态程序不能更改内容也不能删除路由表。
从表中可以看到目的IP为”132.16.88.130"的数据包由接口”eno33554976"来处理,而目的IP为"172.16.96.131"的数据包由接口”eno16777736"来处理。

若数据包目的IP不是本机IP，则在local中匹配不到相应条目，因而根据下一规则32766来查找路由表main。再来看路由表main的内容:
```bash
[root@compute1 ~]# ip route show
default via 172.16.88.2 dev eno33554976
169.254.0.0/16 dev eno16777736  scope link  metric 1002
169.254.0.0/16 dev eno33554976  scope link  metric 1003
172.16.88.0/24 dev eno33554976  proto kernel  scope link  src 172.16.88.130
172.16.96.0/24 dev eno16777736  proto kernel  scope link  src 172.16.96.131
```
在查找路由表时遵循最长匹配原则。比如，表中有两个条目，目的IP网络地址分别为
"172.16.96.0/24"和"172.16.0.0/16", 目的IP为172.16.96.101的数据包两个条目都会匹配，会以匹配的更长的"172.16.96.0/24"条目为准。根据表中内容可以看到，当数据包目标IP属于"172.16.88.0/24"网段时，则从接口"eno33554976”转发。若目标IP属于"172.16.96.0/24"网段，则由接口”eno16777736"转发。若所有目的地址条目都不能匹配，则使用default条目，由接口”eno33554976"转发。

系统所有的路由表保存在文件`/etc/iproute2/rt_tables`中，我们来看一下:
```bash
[root@compute1 ~]# cat /etc/iproute2/rt_tables
#
# reserved values
#
255    local
254    main
253    default
0      unspec
#
# local
#
#1    inr.ruhep
```
若需要添加路由表，只需要将表ID和名字写入该文件中。

系统默认路由的匹配策略是根据目的地址进行匹配。很多情况下这满足不了需求，Linux提供了策略路由的支持，可以基于网络接口，源地址，防火墙标志等其他信息来选择下一跳地址。添加策略路由步骤如下:

* 创建自定义的策略路由表
* 创建路由规则来查找策略路由表
* 在策略路由表中添加路由条目

首先添加一个新的路由表:
```bash
echo “200 custom” >> /etc/iproute2/rt_tables
```
创建查找该路由表的路由规则:
```bash
ip rule add from 192.168.30.200 lookup custom
```
查看现在的路由规则:
```bash
[root@compute1 ~]# ip rule
0:        from all lookup local
32765:    from 192.168.30.200 lookup custom
32766:    from all lookup main
32767:    from all lookup default
```
默认情况下，添加规则使用当前可用的最低优先级。因而后添加的规则优先级会更高。在添加规则时也可以直接指定优先级，如:
```bash
ip rule add iif eno33554976 lookup custom priority 10000
```
再来查看规则内容:
```bash
[root@compute1 ~]# ip rule
0:        from all lookup local
10000:    from all iif eno33554976 lookup custom
32765:    from 192.168.30.200 lookup custom
32766:    from all lookup main
32767:    from all lookup default
```
接下来，在路由表中添加路由条目, 如添加一条默认路由:
```bash
ip route add default via 192.168.30.1 dev eth1 table custom
```
通过规则优先级，对策略路由表没有匹配到相应条目时，更使用默认的main路由表进行处理。

