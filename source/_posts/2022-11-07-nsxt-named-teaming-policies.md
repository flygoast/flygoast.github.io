---
title: NSX-T逻辑交换机配置上行链路绑定策略
date: 2022-11-07 00:21:19
tags:
- NSX-T
categories: Virtualization
---
首先介绍`NSX-T`的基本概念。

参与构建`NSX-T`网络的节点叫做`传输节点(Transport Node)`, 包括`ESXi`主机、`KVM`主机和`EDGE`节点。`传输节点`上需要配置构建`NSX-T`网络所需的`NSX虚拟交换机`，可以新建`N-VDS`类型的交换机，也可以复用`vCenter`上所创建的`VDS`, 如图：
{% img /images/2022-11-07/1.png %}

`逻辑交换机(logical switch)`也叫`分段(segment)`为虚拟机提供网络接入点，它需要附着于`NSX虚拟交换机`之上。有些场景下，`逻辑交换机`并不需要在所有`传输节点`上都存在，`NSX-T`使用`传输区域`来表示`传输节点`的范围。`NSX虚拟交换机`在创建时，需要配置所关联的`传输区域`。

<!--more-->

下图中，`传输节点: ESXi-TN1`上创建的`NSX虚拟交换机`关联了`传输区域: tz1`，`传输节点: ESXi-TN2`上创建的`NSX虚拟交换机`关联了`传输区域: tz1和tz2`, `传输节点: ESXi-TN3`关联了`传输区域: tz2`。这样，`传输区域: tz1`就由`TN1`和`TN2`上的`NSX虚拟交换机`构成，而`传输区域: tz2`就由`TN2`和`TN3`上的`NSX虚拟交换机`构成。
{% img /images/2022-11-07/2.png %}

`逻辑交换机/分段`在创建时需要指定所附着的`传输区域`, 从而限定在`传输节点`上的分布:
{% img /images/2022-11-07/3.png %}

`传输区域`分为`VLAN`类型和`Overlay`类型, 表示不同`传输节点`之间的通信实现方式。`VLAN`类型的`传输区域`不同`传输节点`上`NSX虚拟交换机`之间的通信是基于`VLAN`实现不同的逻辑网络隔离。而`Overlay`类型的`传输区域`的不同`传输节点`上的`NSX虚拟交换机`之间则会建立两两相连的`GENEVE`隧道，基于`GENEVE`的`VNI(Virtual Network Identifier)`来实现不同的逻辑网络隔离。这和`OpenStack`的网络实现类似, 可以参考之前的文章:

* [<<RDO安装all-in-one模式OpenStack虚拟网络架构分析>>](/2017/01/07/all-in-one-neutron/)
* [<<VXLAN原理介绍与实例分析>>](/2017/05/21/vxlan/),
* [<<动态维护FDB表项实现VXLAN通信>>](/2020/04/20/vxlan-fdb/)
* [<<基于BGP EVPN的VXLAN通信实践>>](/2020/04/26/vxlan-evpn/)

`上行链路(uplink)`表示`NSX虚拟交换机`到物理网络的接入点，可以理解为物理网卡的映射。在创建`NSX虚拟交换机`时，需要指定不同的`uplink`接口所映射的实际物理网卡, 如图:
{% img /images/2022-11-07/4.png %}

而如果`NSX虚拟交换机`以`vCenter`上创建的`VDS`为载体，则映射为`VDS`的`uplink`, 如图:
{% img /images/2022-11-07/5.png %}

`上行链路配置文件(uplink profile)`定义了`NSX虚拟交换机`使用`uplink`的策略，这叫做`绑定策略(Teaming Policies)`。默认的策略为`负载均衡源(Load Balance Source)`, 表示根据虚拟机ID均衡分散到不同的`uplink`，也可以修改为`故障切换顺序(Failover Order)`实现主备模式, 或者是`负载均衡源MAC(Load Balance Source MAC)`,表示基于虚拟机网卡的`MAC`地址进行负载均衡。

这种场景下，一个`NSX虚拟交换机`所承载的`逻辑交换机/分段`的`绑定策略`都是`NSX虚拟交换机`创建时所设置的默认`绑定策略`。但有些场景下，不同的`逻辑交换机/分段`需要配置不同的`绑定策略`。比如，一个`NSX虚拟交换机`有`4`个`uplink`，业务网络使用前两个`uplink`构成的`负载均衡源`模式, 而`vMotion`/`vSAN`等管理流量使用另外两个`uplink`构成的`故障切换顺序`模式。

这个需求在`VDS`上配置比较简单，可以针对不同的`端口组`配置不同的`绑定和故障切换`设置:
{% img /images/2022-11-07/6.png %}

在`NSX-T`的`逻辑交换机/分段`上实现该需求步骤则略为复杂一些。

`uplink profile`的`绑定策略`支持配置多个，第一条为默认策略，其他策略则需要指定策略名称，这个机制叫做`
命名绑定策略(Named Teaming Polices)`, 该特性在`NSX-T 2.3`中引入，从`2.4`版本开始，一个`NSX虚拟交换机`可以配置多个`命名绑定策略`, 如图:
{% img /images/2022-11-07/7.png %}

创建`传输区域`时可以关联多个`命名绑定策略`, 如图:
{% img /images/2022-11-07/8.png %}

接下来，基于`传输区域`创建`逻辑交换机`时就可以选择所关联的`命名绑定策略`了，如图:
{% img /images/2022-11-07/9.png %}


将一台虚拟机接入上述所创建`逻辑交换机: ls-uplink-1`。登录虚拟机所在`ESXi`主机，执行:
```bash
esxtop -d2
```
然后按`n`展示`network`界面, 可以看到虚拟机`t1`的网卡绑定的物理网卡为`vmnic0`:
{% img /images/2022-11-07/10.png %}

接着执行命令关闭`vmnic0`:
```bash
esxcli network nic down -n vmnic0
```
再次查看绑定的物理网卡, 发现已经变更为`vmnic1`:
{% img /images/2022-11-07/11.png %}

可以确定`逻辑交换机`的`上行链路绑定策略`的确已经生效。

参考:
* https://wesleygeelhoed.nl/2019/12/16/named-teaming-policy-in-nsx-t-2-5-load-balancing-deep-dive/
* https://virtualrove.com/2020/05/28/nsx-t-3-0-series-part4-create-transport-zones-uplink-profiles/
* https://greatwhitetec.com/2016/11/01/esxtop-not-displaying-properly/
* https://esxsi.com/2016/07/10/esxtop/
* https://tomaskalabis.com/wordpress/vmware-esxi-how-shutdown-vmnic-interface/
* https://vxplanet.com/2019/09/25/achieving-deterministic-peering-using-nsx-t-named-teaming-policies/
* http://www.cloudxtreme.info/nsx-t-uplink-profile/
* https://www.lab2prod.com.au/2022/05/nsx-t-deterministic-traffic-on-vlan-backed-segments.html
