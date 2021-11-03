---
title: Kubernetes POD环境的NFQUEUE机制
date: 2021-10-04 21:01:06
tags:
- network
- kubernetes
- kernel
- nfqueue
categories: Kubernetes
description:
---
之前的文章<<[Kubernetes环境中NFQUEUE与MARK机制冲突](/2021/09/05/nfqueue-mark/)>>介绍了我们使用`NFQUEUE`机制将数据包送往用户态进行安全检测。之前程序逻辑是将来自虚拟网络设备的数据包直接放行。而当把逻辑修改为对`POD`虚拟网卡的流量也进行检测时，`POD`网络就无法连通了。排查发现数据包送上用户态之后，并没有收到用户态程序的裁决信息。

在对`NFQUEUE`的源码实现了进行草略分析后,发现`NFQUEUE`机制是支持`network namespace`的。`POD`虚拟网络设备的数据包送往用户态的队列是在`POD`独有的`network namespace`中创建的，和默认的`network namespace`:`init_net`中的队列是完全独立的。我们的用户态程序运行是在`init_net`中运行，而`POD`的`network namespace`中并没有用户态程序在读取队列获取数据包，因而数据包会被丢弃。

和之前文章同样，通过简化程序来进行实验。实验的`Kubernetes`环境有3个`node`, 容器组网使用`flannel`。

我们创建了两个`busybox`的`pod`：
```bash
kubectl run busybox1 --image=busybox --command -- sleep 3600
kubectl run busybox2 --image=busybox --command -- sleep 3600
```

他们分别位于`node1`和`node2`上:
```plain
[root@master1 scripts]# kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
busybox1-77bb94599d-x89z4   1/1     Running   12         22h   10.230.96.4    node1   <none>           <none>
busybox2-7d76b658b6-h5r2k   1/1     Running   10         22h   10.230.12.2    node2   <none>           <none>
```

我们从`busybox2`中访问`busybox1`，网络连通正常:
```plain
[root@master1 scripts]# kubectl exec -ti busybox2-7d76b658b6-h5r2k -- ping -c2 10.230.96.4
PING 10.230.96.4 (10.230.96.4): 56 data bytes
64 bytes from 10.230.96.4: seq=0 ttl=62 time=1.076 ms
64 bytes from 10.230.96.4: seq=1 ttl=62 time=0.770 ms

--- 10.230.96.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.770/0.923/1.076 ms
```

<!--more-->

编写一个内核模块将数据包通过`NFQUEUE`队列送往用户态:
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/skbuff.h>
#include <linux/ip.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h>
#include <linux/inet.h>

MODULE_LICENSE("GPL");

static int queue_bypass = 0;
MODULE_PARM_DESC(clear_mark, "queue bypass on/off switch");
module_param(queue_bypass, int, 0600);

static unsigned int nf_hook_in1(const struct nf_hook_ops *ops,
                   struct sk_buff *skb,
                   const struct net_device *in,
                   const struct net_device *out,
                   const struct nf_hook_state *state)
{
    unsigned int ret;
    struct iphdr *iph;

    if (skb->protocol != __constant_htons(ETH_P_IP)) {
        return NF_ACCEPT;
    }

    iph = ip_hdr(skb);
    if (iph && iph->protocol != IPPROTO_ICMP) {
        return NF_ACCEPT;
    }

    printk(KERN_INFO "NSQUEUE: ICMP packet: %u.%u.%u.%u->%u.%u.%u.%u, " \
            "init_net: %p, net: %p, devname: %s\n",
            (ntohl(iph->saddr) >> 24) & 0xFF,
            (ntohl(iph->saddr) >> 16) & 0xFF,
            (ntohl(iph->saddr) >> 8)  & 0xFF,
            (ntohl(iph->saddr))       & 0xFF,
            (ntohl(iph->daddr) >> 24) & 0xFF,
            (ntohl(iph->daddr) >> 16) & 0xFF,
            (ntohl(iph->daddr) >> 8)  & 0xFF,
            (ntohl(iph->daddr))       & 0xFF,
            &init_net,
            dev_net(skb->dev),
            skb->dev->name
            );

    ret = NF_QUEUE_NR(80);
    if (queue_bypass) {
        ret |= NF_VERDICT_FLAG_QUEUE_BYPASS;
    }

    return ret;
}

static struct nf_hook_ops nfhooks[] = {
{
    .hook = nf_hook_in1,
    .owner = THIS_MODULE,
    .pf = NFPROTO_IPV4,
    .hooknum = NF_INET_LOCAL_IN,
    .priority = NF_IP_PRI_FIRST,
},
};

int __init nsqueue_init(void)
{
    nf_register_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    printk(KERN_INFO "NSQUEUE: module init\n");

    return 0;
}

void __exit nsqueue_exit(void)
{
    nf_unregister_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    printk(KERN_INFO "NSQUEUE: module exit\n");

    return;
}

module_init(nsqueue_init);
module_exit(nsqueue_exit);
```

我们将`ICMP`的数据包通过`NFQUEUE`机制送往`80`号队列。

`Makefile`:
```makefile
obj-m += nsqueue.o
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

在`busybox1`所在的`node1`上编译并装载`nsqueue.ko`:
```bash
insmod nsqueue.ko
```

然后我们再次尝试从`busybox2`访问`busybox1`, 访问超时, 网络确实断了:
```plain
[root@master1 scripts]# kubectl exec -ti busybox2-7d76b658b6-h5r2k -- ping -c2 10.230.96.4
PING 10.230.96.4 (10.230.96.4): 56 data bytes

--- 10.230.96.4 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
command terminated with exit code 1
```

查看`node1`的`dmesg`信息, 可以看到`nsqueue.ko`处理了两个数据包:
```plain
[32555.762163] NSQUEUE: module init
[32634.888279] NSQUEUE: ICMP packet: 10.230.12.0->10.230.96.4, init_net: ffffffff85315bc0, net: ffff8ac077451480, devname: eth0
[32635.888896] NSQUEUE: ICMP packet: 10.230.12.0->10.230.96.4, init_net: ffffffff85315bc0, net: ffff8ac077451480, devname: eth0
```

接下来我们来在`busybox1`的`network namespace`中运行用户态程序。

首先，通过`kubectl`获取`container id`:
```plain
[root@master1 scripts]# kubectl describe pod busybox1-77bb94599d-x89z4 |grep 'Container ID'
    Container ID:  containerd://4872c95767c2504f6d646b54e6843a30905d0dec27b9d5934d01f3301ac220e1
```

接着在`node1`上通过`container id`找到`pod`的`pid`:
```plain
[root@node1 vagrant]# crictl inspect 4872c95767c2504f6d646b54e6843a30905d0dec27b9d5934d01f3301ac220e1 |grep pid
    "pid": 25207,
            "pid": 1
            "type": "pid"
```

然后在`node1`上通过`nsenter`进入`busybox1`的`network namespace`:
```plain
[root@node1 vagrant]# nsenter -n -t 25207
[root@node1 vagrant]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether c2:1a:09:a7:69:78 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.230.96.4/24 brd 10.230.96.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c01a:9ff:fea7:6978/64 scope link
       valid_lft forever preferred_lft forever
```

我们的用户态程序还是使用[之前文章](/2021/09/05/nfqueue-mark/)中的代码。在`busybox1`的`network namespace`运行用户态程序`a.out`:
```plain
[root@node1 vagrant]# ./a.out
```

再次尝试`pod`间的访问, 可以看到又可以连通了:
```plain
[root@master1 scripts]# kubectl exec -ti busybox2-7d76b658b6-h5r2k -- ping -c2 10.230.96.4
PING 10.230.96.4 (10.230.96.4): 56 data bytes
64 bytes from 10.230.96.4: seq=0 ttl=62 time=8.954 ms
64 bytes from 10.230.96.4: seq=1 ttl=62 time=2.059 ms

--- 10.230.96.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 2.059/5.506/8.954 ms
```

查看`a.out`的输出，可以看到两个数据包被用户态程序处理:
```c
[root@node1 vagrant]# ./a.out
packet: 1, origin mark: 0x00000000, set mark: 0x00000000
packet: 2, origin mark: 0x00000000, set mark: 0x00000000
```

我们的内核模块代码中带有一个`queue_bypass`参数，当参数被设置时，`netfilter`的`hook`函数返回值会附带`NF_VERDICT_FLAG_QUEUE_BYPASS`标志。在附带这个标志时，内核协议栈会在队列不存在时跳到下一个`hook`函数而不是丢弃该数据包。

我们将`queue_bypass`参数修改为`1`:
```bash
echo 1 > /sys/module/nsqueue/parameters/queue_bypass
```

在不运行用户态程序的情况下再次尝试`POD`间访问，成功连通:
```plain
[root@master1 scripts]# kubectl exec -ti busybox2-7d76b658b6-h5r2k -- ping -c2 10.230.96.4
PING 10.230.96.4 (10.230.96.4): 56 data bytes
64 bytes from 10.230.96.4: seq=0 ttl=62 time=5.606 ms
64 bytes from 10.230.96.4: seq=1 ttl=62 time=3.708 ms

--- 10.230.96.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 3.708/4.657/5.606 ms
```
