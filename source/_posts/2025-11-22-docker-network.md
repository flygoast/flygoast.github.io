---
title: Docker容器数据包转发路径分析
date: 2025-11-22 21:29:26
tags:
- Network
- Docker
- Kernel
- Netfilter
categories: Kernel
---
这些年总结过多次容器网络数据包路径相关的文章, 如:
* [从外部访问Docker桥接网络容器路径分析](/2023/08/24/docker-bridge/)
* [基于netfilter防护Docker容器网络](/2024/12/05/docker-netfilter/)
* [kolla部署openstack环境中的异常NAT](/2024/12/25/kolla-openstack-nat/)
* [相同网桥上的网络隔离](/2025/03/19/bridge-isolation/)

但每次过段时间后，总会忘记细节。分析容器网络异常是否是由于某些基于`netfilter`的驱动影响时，总是要重新梳理。这次再从内核实现的角度来分析一次容器网络数据包的转发路径。

还是以外部访问`bridge`模式`docker`容器的场景进行分析。

#### 入包

外部主机访问`docker`容器的数据包到达网卡后, 由内核函数:`netif_receive_skb`处理进入协议栈处理。对于`IP`数据包，会调用到函数`ip_rcv`：
```c
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, NULL, skb,
               dev, NULL,
               ip_rcv_finish);
```

在这里会进行`netfilter`的`PRE_ROUTING`阶段处理。入包在`PRE_ROUTING`阶段会由`docker`的`iptables`规则完成`DNAT`操作，数据包目的`IP`变更为`docker`容器的IP。

<!--more-->

之后调用到函数`ip_rcv_finish`。`ip_rcv_finish`会进行路由查找，接着通过`dst_input`函数，调用路由中指定的函数。对于发送到本机的数据包，会调用`ip_local_deliver`交由传输层处理。对于访问`docker`容器这个场景，路由指向`docker0`, 调用`ip_forward`处理:
```c
    return NF_HOOK(NFPROTO_IPV4, NF_INET_FORWARD, NULL, skb,
               skb->dev, rt->dst.dev, ip_forward_finish);
```

这里进行`netfilter`的`FORWARD`阶段处理, 之后调用到函数:`ip_forward_finish`。`ip_forward_finish`最终会调用`dst_output_sk`,  它调用到`ip_output`:
```c
int ip_output(struct sock *sk, struct sk_buff *skb)
{
    struct net_device *dev = skb_dst(skb)->dev;

    IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUT, skb->len);

    skb->dev = dev;
    skb->protocol = htons(ETH_P_IP);

    return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, sk, skb,
                NULL, dev,
                ip_finish_output,
                !(IPCB(skb)->flags & IPSKB_REROUTED));
}
```
这里进行`netfilter`的`POST_ROUTING`阶段处理。最终由`ip_finish_output`调用到`dev_queue_xmit`从`docker0`设备发出。

`dev_queue_xmit`函数中发送会调用`net_device`设备的`net_device_ops`中的`ndo_start_xmit`指针。`docker0`是`bridge`设备接口，它的`ndo_start_xmit`指向`br_dev_xmit`。于是函数调用到`br_dev_xmit`，数据包从路由转发进入到网桥上进行二层转发。

`br_dev_xmit`函数会根据目标`MAC`地址，决策由哪个端口转发，调用`br_forward`转发数据包。它会再调用`__br_forward`函数:
```c
    NF_HOOK(NFPROTO_BRIDGE, br_hook,
        NULL, skb, indev, skb->dev,
        br_forward_finish);
```

这里进行`netfilter`的二层过滤的`NF_BR_LOCAL_OUT`阶段的处理。它表示数据包来自主机的上层协议栈。

最后调用`br_forward_finish`, 在这里进行`netfilter`的二层过滤的`NF_BR_POST_ROUTING`阶段处理:
```c
int br_forward_finish(struct sock *sk, struct sk_buff *skb)
{
    return NF_HOOK(NFPROTO_BRIDGE, NF_BR_POST_ROUTING, sk, skb,
               NULL, skb->dev,
               br_dev_queue_push_xmit);

}
```
`br_netfilter`函数在`NF_BR_POST_ROUTING`注册了函数`br_nf_post_routing`:
```c
    NF_HOOK(pf, NF_INET_POST_ROUTING, state->sk, skb,
        NULL, realoutdev,
        br_nf_dev_queue_xmit);

    return NF_STOLEN;
```

当参数`net.bridge.bridge-nf-call-iptables`开启时，会将二层数据包送到三层进行过滤。

`br_dev_queue_push_xmit`调用`dev_queue_xmit`从网桥上的端口发出。这时目标端口是`veth pair`。`veth pair`设备的`ndo_start_xmit`指针指向`veth_xmit`函数, 它会调用`dev_forward_skb`将数据包由对端接收:
```c
int dev_forward_skb(struct net_device *dev, struct sk_buff *skb)
{
    return __dev_forward_skb(dev, skb) ?: netif_rx_internal(skb);
}
EXPORT_SYMBOL_GPL(dev_forward_skb);
```
这会调用`netif_rx_internal`函数，这时数据就进入到容器`network namespace`, 开始在容器内收包流程。容器`network namespace`中的路由表判断是发送给自己，于是上交到传输层处理，最终被应用层的`socket`接收。

整体的调用链如下:
```plain
netif_receive_skb
--ip_rcv
----NF_INET_PRE_ROUTING
----ip_rcv_finish
------dst_input
--------ip_forward
----------NF_INET_FORWARD
----------ip_forward_finish
------------dst_output_sk
--------------ip_output
----------------NF_INET_POST_ROUTING
----------------ip_finish_output
------------------dev_queue_xmit
--------------------br_dev_xmit-----------------------进入桥接层
----------------------__br_forward
------------------------NF_BR_LOCAL_OUT
------------------------br_forward_finish
--------------------------NF_BR_POST_ROUTING
----------------------------br_nf_post_routing---------三层过滤
--------------------------br_dev_queue_push_xmit
----------------------------dev_queue_xmit
------------------------------veth_xmit
--------------------------------dev_forward_skb
----------------------------------net_if_rx_internal------------进入容器
```

#### 回包:

回包从容器`network namespace`的网卡由`dev_queue_xmit`发送，容器内网卡为`veth pair`设备，由`veth_xmit`调用触发主机上的`veth pair`设备的收包逻辑, 调用到函数`__netif_receive_skb_core`
。此时，数据包进入到宿主机`network namespace`。

由于`veth pair`设备挂接到`bridge`设备，它的`net_device->rx_handler`会被设置为`br_handle_frame`。`__netif_receive_skb_core`于是调用到`br_handle_frame`函数:
```c
    rx_handler = rcu_dereference(skb->dev->rx_handler);
    if (rx_handler) {
        if (pt_prev) {
            ret = deliver_skb(skb, pt_prev, orig_dev);
            pt_prev = NULL;
        }
        switch (rx_handler(&skb)) {
        case RX_HANDLER_CONSUMED:
            ret = NET_RX_SUCCESS;
            goto out;
        case RX_HANDLER_ANOTHER:
            goto another_round;
        case RX_HANDLER_EXACT:
            deliver_exact = true;
        case RX_HANDLER_PASS:
            break;
        default:
            BUG();
        }
    }
```
`br_handle_frame`根据目的`MAC`地址决策目的端口转发。
```c
forward:
    switch (p->state) {
    case BR_STATE_FORWARDING:
        rhook = rcu_dereference(br_should_route_hook);
        if (rhook) {
            if ((*rhook)(skb)) {
                *pskb = skb;
                return RX_HANDLER_PASS;
            }
            dest = eth_hdr(skb)->h_dest;
        }
        /* fall through */
    case BR_STATE_LEARNING:
        if (ether_addr_equal(p->br->dev->dev_addr, dest))
            skb->pkt_type = PACKET_HOST;

        NF_HOOK(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING, NULL, skb,
            skb->dev, NULL,
            br_handle_frame_finish);
        break;
    default:
drop:
        kfree_skb(skb);
    }
```

这里进行`netfilter`的二层过滤的`NF_BR_PRE_ROUTING`阶段的处理。

`br_netfilter`内核模块在这个阶段注册了函数: `br_nf_pre_routing`, 当参数`net.bridge.bridge-nf-call-iptables`开启时，会将二层数据包送到三层过滤:
```c
    NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, state->sk, skb,
        skb->dev, NULL,
        br_nf_pre_routing_finish);
```


`br_handle_frame`最终调用`br_handle_frame_finish`进行转发决策。此时回包`MAC`为`docker0`, 是网桥自身的三层接口，这种情况下，并不会由其他端口转发，而是调用函数`br_pass_frame_up`上送协议栈处理：
```c
    return NF_HOOK(NFPROTO_BRIDGE, NF_BR_LOCAL_IN, NULL, skb,
               indev, NULL,
               br_netif_receive_skb);
```
这里进行`NF_BR_LOCAL_IN`阶段的处理。表示数据由二层进入三层协议栈。

`br_netif_receive_skb`调用`netif_receive_skb`进入宿主机的收包阶段:
```c
static int br_netif_receive_skb(struct sock *sk, struct sk_buff *skb)
{
    br_drop_fake_rtable(skb);
    return netif_receive_skb(skb);
}
```

接着同入包流程类似，会调用到`ip_rcv`函数，进入宿主机上`netfilter`的`PRE_ROUTING`阶段处理。接着根据路由结果由`ip_forward`进行转发，进行`netfilter`的`FORWARD`阶段的处理。最终由`ip_forward_finish`调用`ip_output`, 在这里进行`POST_ROUTING`阶段的处理。容器回包在这里进行`SNAT`操作，将源`IP`修改为宿主机`IP`，然后由物理网卡发出。

整体调用链为:
```plain
dev_queue_xmit
--veth_xmit
----__netif_receive_skb_core
------br_handle_frame
--------NF_BR_PRE_ROUTING
--------br_handle_frame_finish
----------br_pass_frame_up
------------NF_BR_LOCAL_IN
------------br_netif_receive_skb
--------------netif_receive_skb
----------------ip_rcv
------------------NF_INET_PRE_ROUTING
------------------ip_rcv_finish
--------------------dst_input
----------------------ip_forward
------------------------NF_INET_FORWARD
------------------------ip_forward_finish
--------------------------dst_output_sk
----------------------------ip_output
------------------------------NF_INET_POST_ROUTING
------------------------------ip_finish_output
--------------------------------dev_queue_xmit
```

容器之间访问的数据包会由函数`__br_forward`处理，这时会进行`netfilter`的二层过滤`NF_BR_FORWARD`阶段处理。如果开启参数`
net.bridge.bridge-nf-call-iptables`, 会将数据包送到三层进行过滤。
