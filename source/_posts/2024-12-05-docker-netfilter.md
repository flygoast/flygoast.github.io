---
title: 基于netfilter防护Docker容器网络
date: 2024-12-05 16:17:00
tags:
- Network
- Kernel
categories: Kernel
---
我们的主机网络防护是基于`netfilter`实现。最近遇到需要对访问主机上`Docker`容器的流量进行防护。几年前其实就处理过这个场景，时间久远忘记了，重新梳理一下记录下来。

我们的主机网络防护模块的`hooknum`为`LOCAL_IN`和`POST_ROUTING`, 并且`hook`的优先级为`NF_IP_PRI_FIRST`, 也就是在`hooknum`位置最先运行。

#### 从宿主机外部访问主机上容器的场景

之前的文章[<<从外部访问Docker桥接网络容器路径分析>>](/2023/08/24/docker-bridge/)分析了从外部访问`Docker`桥接网络的网络路径。

*   从外部到容器的数据包会流经`PRE_ROUTING`, `FORWARD`和 `POST_ROUTING`阶段，在`PRE_ROUTING`阶段会进行`DNAT`, 将目的`IP/PORT`, 修改为容器的`IP/PORT`。 
*   从容器到外部的数据包会流经`PRE_ROUTING`, `FORWARD`和`POST_ROUTING`阶段，在`POST_ROUTING`阶段会进行`SNAT`, 将源`IP/PORT`修改为外部宿主机的`IP/PORT`。

从网络路径来看，在`POST_ROUTING`阶段数据包上的地址是容器本身的地址, 因而我们可以简单的将容器`IP/PORT`端口在规则中配置，就可以实现对于访问容器内部流量的防护。

<!--more-->

可以通过内核模块来验证这种场景下的一下, 源码如下:

```text-x-csrc
#define pr_fmt(fmt) "[%s]: " fmt, KBUILD_MODNAME

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/skbuff.h>
#include <linux/ip.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <net/tcp.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h>
#include <linux/inet.h>
#include <net/checksum.h>

static unsigned int nf_hook_trace(
        const struct nf_hook_ops *ops,
        struct sk_buff *skb,
        const struct net_device *in,
        const struct net_device *out,
        int (*okfn)(struct sk_buff *))
{
    struct tcphdr *tcph = NULL;
    struct iphdr *iph = ip_hdr(skb);

    u8 proto = iph->protocol;

    if (proto != IPPROTO_TCP) {
        return NF_ACCEPT;
    }

    tcph = (struct tcphdr *) skb_transport_header(skb);

    if (htons(tcph->dest) == 8080 || htons(tcph->source) == 8080 ||
        htons(tcph->dest) == 8081 || htons(tcph->source) == 8081 )
    {
        pr_info("====================================================");

        pr_info("DEV: %s, NS: %p, INITNET: %p",
            skb->dev ? skb->dev->name : "(no dev)",
            skb->dev ? skb->dev->nd_net : NULL,
            &init_net);

        pr_info("IN DEV: %s, NS: %p, OUT DEV: %s, NS: %p",
            in ? in->name : "(null)", in ? in->nd_net : NULL,
            out ? out->name : "(null)", out ? out->nd_net : NULL);

        pr_info("[%d]: %pI4:%d->%pI4:%d, skb: %p\n", ops->hooknum,
            &iph->saddr, htons(tcph->source), &iph->daddr, htons(tcph->dest), skb);
        return NF_ACCEPT;
    }

    return NF_ACCEPT;
}

static struct nf_hook_ops nfhooks[] = {
    {
        .hook = nf_hook_trace,
        .owner = THIS_MODULE,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_PRE_ROUTING,
        .priority = NF_IP_PRI_FIRST,
    },
    {
        .hook = nf_hook_trace,
        .owner = THIS_MODULE,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_LOCAL_IN,
        .priority = NF_IP_PRI_FIRST,
    },
    {
        .hook = nf_hook_trace,
        .owner = THIS_MODULE,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_FORWARD,
        .priority = NF_IP_PRI_FIRST,
    },
    {
        .hook = nf_hook_trace,
        .owner = THIS_MODULE,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_POST_ROUTING,
        .priority = NF_IP_PRI_FIRST,
    },
    {
        .hook = nf_hook_trace,
        .owner = THIS_MODULE,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_LOCAL_OUT,
        .priority = NF_IP_PRI_FIRST,
    },
};

int __init nftrace_init(void)
{
    nf_register_hooks(nfhooks, ARRAY_SIZE(nfhooks));
    pr_info("nftrace module init\n");
    return 0;
}

void __exit nftrace_exit(void)
{
    nf_unregister_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    pr_info("nftrace module exit\n");
    return;
}

module_init(nftrace_init);
module_exit(nftrace_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("nftrace"); 
```

测试环境中，容器内部监听`8080`， 宿主机上监听`8081`转发到容器内的`8080`。在另外一台机器访问宿主机的`8081`端口，查看内核模块的输出结果。

入数据包的结果是:

```text-plain
[3598646.157916] [nftrace]: ====================================================
[3598646.157920] [nftrace]: DEV: ens160, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3598646.157931] [nftrace]: IN DEV: ens160, NS: ffffffff8d911640, OUT DEV: (null), NS:           (null)
[3598646.157934] [nftrace]: [0]: 10.10.10.3:38478->10.10.10.2:8081, skb: ffff9710d3f58d00
[3598646.157950] [nftrace]: ====================================================
[3598646.157951] [nftrace]: DEV: ens160, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3598646.157954] [nftrace]: IN DEV: ens160, NS: ffffffff8d911640, OUT DEV: docker0, NS: ffffffff8d911640
[3598646.157955] [nftrace]: [2]: 10.10.10.3:38478->172.17.0.2:8080, skb: ffff9710d3f58d00
[3598646.157960] [nftrace]: ====================================================
[3598646.157961] [nftrace]: DEV: docker0, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3598646.157963] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: docker0, NS: ffffffff8d911640
[3598646.157964] [nftrace]: [4]: 10.10.10.3:38478->172.17.0.2:8080, skb: ffff9710d3f58d00
```

因为在`CentOS7`的`3.10`版本内核上，注册`netfilter`函数会在所有的`network namespace`上生效，因而也会有在容器本身的`namespace`中独立的`netfilter`流经过程:

```text-plain
[3598646.157983] [nftrace]: ====================================================
[3598646.157984] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3598646.157987] [nftrace]: IN DEV: eth0, NS: ffff9710acf80000, OUT DEV: (null), NS:           (null)
[3598646.157988] [nftrace]: [0]: 10.10.10.3:38478->172.17.0.2:8080, skb: ffff9710d3f58d00
[3598646.157997] [nftrace]: ====================================================
[3598646.157998] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3598646.158000] [nftrace]: IN DEV: eth0, NS: ffff9710acf80000, OUT DEV: (null), NS:           (null)
[3598646.158001] [nftrace]: [1]: 10.10.10.3:38478->172.17.0.2:8080, skb: ffff9710d3f58d00
[3598646.158012] [nftrace]: ====================================================
[3598646.158013] [nftrace]: DEV: (no dev), NS:           (null), INITNET: ffffffff8d911640
[3598646.158015] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf80000
[3598646.158017] [nftrace]: [3]: 172.17.0.2:8080->10.10.10.3:38478, skb: ffff9710787a8500
[3598646.158020] [nftrace]: ====================================================
[3598646.158021] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3598646.158024] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf80000
[3598646.158025] [nftrace]: [4]: 172.17.0.2:8080->10.10.10.3:38478, skb: ffff9710787a8500
```

容器响应包的结果:

```text-plain
[3598646.158033] [nftrace]: ====================================================
[3598646.158034] [nftrace]: DEV: docker0, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3598646.158037] [nftrace]: IN DEV: docker0, NS: ffffffff8d911640, OUT DEV: (null), NS:           (null)
[3598646.158038] [nftrace]: [0]: 172.17.0.2:8080->10.10.10.3:38478, skb: ffff9710787a8500
[3598646.158042] [nftrace]: ====================================================
[3598646.158043] [nftrace]: DEV: docker0, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3598646.158045] [nftrace]: IN DEV: docker0, NS: ffffffff8d911640, OUT DEV: ens160, NS: ffffffff8d911640
[3598646.158046] [nftrace]: [2]: 172.17.0.2:8080->10.10.10.3:38478, skb: ffff9710787a8500
[3598646.158048] [nftrace]: ====================================================
[3598646.158049] [nftrace]: DEV: ens160, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3598646.158051] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: ens160, NS: ffffffff8d911640
[3598646.158052] [nftrace]: [4]: 172.17.0.2:8080->10.10.10.3:38478, skb: ffff9710787a8500
```

#### 主机容器之间访问

我们接着用上边的内核模块分析这种场景。我们从容器`172.17.0.4`访问`172.17.0.2:8080`。

请求包结果:

```text-plain
[3612381.406210] [nftrace]: ==================================================== LOCAL_OUT
[3612381.406214] [nftrace]: DEV: (no dev), NS:           (null), INITNET: ffffffff8d911640
[3612381.406217] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf85200
[3612381.406219] [nftrace]: [3]: 172.17.0.4:41050->172.17.0.2:8080, skb: ffff9710e5e504f8
[3612381.406230] [nftrace]: ==================================================== POST_ROUTING
[3612381.406231] [nftrace]: DEV: eth0, NS: ffff9710acf85200, INITNET: ffffffff8d911640
[3612381.406233] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf85200
[3612381.406234] [nftrace]: [4]: 172.17.0.4:41050->172.17.0.2:8080, skb: ffff9710e5e504f8
[3612381.406252] [nftrace]: ==================================================== PRE_ROUTING
[3612381.406253] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3612381.406255] [nftrace]: IN DEV: eth0, NS: ffff9710acf80000, OUT DEV: (null), NS:           (null)
[3612381.406256] [nftrace]: [0]: 172.17.0.4:41050->172.17.0.2:8080, skb: ffff9710e5e504f8
[3612381.406263] [nftrace]: ==================================================== LOCAL_IN
[3612381.406264] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3612381.406265] [nftrace]: IN DEV: eth0, NS: ffff9710acf80000, OUT DEV: (null), NS:           (null)
[3612381.406266] [nftrace]: [1]: 172.17.0.4:41050->172.17.0.2:8080, skb: ffff9710e5e504f8
```

首先，请求数据包流经的是容器`172.17.0.4`所在`namespace`的`LOCAL_OUT`和`POST_ROUTING`阶段，然后直接进入到`172.17.0.2`容器所在`namespace`的`PRE_ROUTING`和`LOCAL_IN`阶段。

响应包也类似:

```text-plain
[3612381.406274] [nftrace]: ==================================================== LOCAL_OUT
[3612381.406275] [nftrace]: DEV: (no dev), NS:           (null), INITNET: ffffffff8d911640
[3612381.406277] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf80000
[3612381.406278] [nftrace]: [3]: 172.17.0.2:8080->172.17.0.4:41050, skb: ffff970d65f34000
[3612381.406281] [nftrace]: ==================================================== POST_ROUTING
[3612381.406281] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3612381.406282] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf80000
[3612381.406283] [nftrace]: [4]: 172.17.0.2:8080->172.17.0.4:41050, skb: ffff970d65f34000
[3612381.406293] [nftrace]: ==================================================== PRE_ROUTING
[3612381.406294] [nftrace]: DEV: eth0, NS: ffff9710acf85200, INITNET: ffffffff8d911640
[3612381.406295] [nftrace]: IN DEV: eth0, NS: ffff9710acf85200, OUT DEV: (null), NS:           (null)
[3612381.406296] [nftrace]: [0]: 172.17.0.2:8080->172.17.0.4:41050, skb: ffff970d65f34000
[3612381.406299] [nftrace]: ==================================================== LOCAL_IN
[3612381.406299] [nftrace]: DEV: eth0, NS: ffff9710acf85200, INITNET: ffffffff8d911640
[3612381.406300] [nftrace]: IN DEV: eth0, NS: ffff9710acf85200, OUT DEV: (null), NS:           (null)
[3612381.406301] [nftrace]: [1]: 172.17.0.2:8080->172.17.0.4:41050, skb: ffff970d65f34000
```

依次是`172.17.0.2`容器所在`namespace`的`LOCAL_OUT`、`POST_ROUTING`和`172.17.0.2`容器所在的`namespace`的`PRE_ROUTING`和`LOCAL_IN`阶段。

并没有流经宿主机所在的`init_net`的`netfilter`阶段。

这是因为同一主机上不同容器之间的流量是通过`Linux bridge`进行二层转发，默认情况下，不会调到到三层的`netfilter`框架。但内核中有参数可以调整该行为，默认是关闭的。

```text-plain
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
```

将`net.bridge.bridge-nf-call-iptables`参数设为`1`后，再次进行相同的测试, 结果如下:

```text-plain
[3615588.895871] [nftrace]: ====================================================
[3612130.508326] [nftrace]: ==================================================== LOCAL_OUT
[3612130.508329] [nftrace]: DEV: (no dev), NS:           (null), INITNET: ffffffff8d911640
[3612130.508332] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf85200
[3612130.508334] [nftrace]: [3]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8
[3612130.508343] [nftrace]: ==================================================== POST_ROUTING
[3612130.508345] [nftrace]: DEV: eth0, NS: ffff9710acf85200, INITNET: ffffffff8d911640
[3612130.508346] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: eth0, NS: ffff9710acf85200
[3612130.508347] [nftrace]: [4]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8

[3612130.508369] [nftrace]: ==================================================== PRE_ROUTING
[3612130.508371] [nftrace]: DEV: docker0, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3612130.508372] [nftrace]: IN DEV: docker0, NS: ffffffff8d911640, OUT DEV: (null), NS:           (null)
[3612130.508373] [nftrace]: [0]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8
[3612130.508382] [nftrace]: ====================================================
[3612130.508383] [nftrace]: DEV: vethfbb20cf, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3612130.508385] [nftrace]: IN DEV: docker0, NS: ffffffff8d911640, OUT DEV: docker0, NS: ffffffff8d911640
[3612130.508386] [nftrace]: [2]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8
[3612130.508389] [nftrace]: ====================================================
[3612130.508390] [nftrace]: DEV: vethfbb20cf, NS: ffffffff8d911640, INITNET: ffffffff8d911640
[3612130.508391] [nftrace]: IN DEV: (null), NS:           (null), OUT DEV: docker0, NS: ffffffff8d911640
[3612130.508392] [nftrace]: [4]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8

[3612130.508398] [nftrace]: ==================================================== PRE_ROUTING
[3612130.508399] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3612130.508400] [nftrace]: IN DEV: eth0, NS: ffff9710acf80000, OUT DEV: (null), NS:           (null)
[3612130.508401] [nftrace]: [0]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8
[3612130.508407] [nftrace]: ==================================================== LOCAL_IN
[3612130.508408] [nftrace]: DEV: eth0, NS: ffff9710acf80000, INITNET: ffffffff8d911640
[3612130.508409] [nftrace]: IN DEV: eth0, NS: ffff9710acf80000, OUT DEV: (null), NS:           (null)
[3612130.508410] [nftrace]: [1]: 172.17.0.4:41052->172.17.0.2:8080, skb: ffff970f9fe2c0f8
```

从结果分析，数据包也流经了`init_net`的`PRE_ROUTING`、`FORWARD`和`POST_ROUTING`阶段。回包也是一样的。因此，对于容器间的访问，我们也可以在`POST_ROUTING`阶段进行规则的配置来实现容器防护需求。

其实，从上述的结果看，如果不基于现有的主机防护能力，其实最理想的位置是`FORWARD`位置，这个位置的入设备和出设备都已经在参数中明确。如果只是在`netfilter`中实现过滤，在容器本身的`namespace`中也可以实现流量过滤，但如果需要使用`NFQUEUE`将数据包上送到用户态处理，那就需要所有的容器`namespace`中都运行收包检测程序，这种方案就不太适用了。之前的文章[<<Kubernetes POD环境的NFQUEUE机制>>](/2021/10/04/nfqueue-pod/)分析过这种情况。

参考:

*   [https://blog.csdn.net/flxzlxb/article/details/112849468](https://blog.csdn.net/flxzlxb/article/details/112849468)
*   [https://cloud.tencent.com/developer/article/1982151](https://cloud.tencent.com/developer/article/1982151)
