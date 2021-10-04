---
title: Kubernetes环境中NFQUEUE与MARK机制冲突
date: 2021-09-05 22:03:23
tags:
- network
- kubernetes
- kernel
categories: Network
description:
---
在Kubernetes节点上安装我们的流量检测模块之后所有的`Pod`会断网。经分析是由于流量检测模块的`NFQUEUE`机制与`kube-proxy`使用的`iptables`的`mark`机制冲突的原因。

在Linux内核中，网络数据包是由`sk_buff`结构来表示的，一般数据包简写作`SKB`。`mark`是`sk_buff`结构的一个字段, 如(`include/linux/skbuff.h`):

```c
struct sk_buff {
    ...
	union {
		__u32		mark;
		__u32		reserved_tailroom;
	};
    ...
}
```

`mark`并不是网络协议结构的部分，不会存在于任一层协议头中，而是Linux网络子系统用于在主机内部传递状态信息的标记机制。各种网络应用可以根据自身需要使用该字段来实现自身的状态传递。

这个`mark`机制主要用在`netfilter`框架中，所以也叫`nfmark`。除了它之外，内核中还有`conntrack`模块也有自己的`mark`机制，一般叫做`ctmark`。

之前的文章[<<基于IPTABLES MARK机制实现策略路由>>](/2016/12/23/iptables-mark-and-polices-based-route/)也介绍过`iptables`的`MARK`模块，可以用于修改和匹配数据包的`mark`值。

`NFQUEUE`机制可以在内核中将数据通过`NFQUEUE`通道将数据包送往用户态，在用户态进行安全检测，再将裁决(`verdict`)结果送回内核。之前的文章[<<NFQUEUE和libnetfilter_queue实例分析>>](/2017/03/27/nfqueue/)介绍了`libnetfilter_queue`库的简单用法。我们的流量检测程序会使用`libnetfilter_queue`库中的`nfq_set_verdict2`在返回`verdict`的同时，设置数据包的`mark`值,以传递更多的信息给内核模块，函数原型如下:
```c
int nfq_set_verdict2(struct nfq_q_handle *  qh,
                     uint32_t               id,
                     uint32_t               verdict,
                     uint32_t               mark,
                     uint32_t               data_len,
                     const unsigned char*   buf
)
```
这就会导致数据包`sk_buff`结构的`mark`值被设置。而`kube-proxy`实现也依赖`iptables`的`mark`机制, 会在主机上添加如下`iptables`规则:
```
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
...
-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
```
对于不合法的报文，`kube-proxy`会给相应报文标记`0x8000/0x8000`, 之后通过`KUBE-FIREWALL`规则链将数据包丢弃。

如果我们的流量检测程序所设置的`mark`值设置为`kube-proxy`所依赖的`0x8000`位，就会导致数据包被丢弃。

<!--more-->

下面通过简化后的程序来验证。

首先，编写一个内核模块将数据包通过`NFQUEUE`队列送往用户态:
```
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

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("skpid");
MODULE_ALIAS("module skpid netfiler");

static int clear_mark = 0;
MODULE_PARM_DESC(clear_mark, "clear sk_buff mark on/off switch");
module_param(clear_mark, int, 0600);

static unsigned int nf_hook_in1(unsigned int hooknum,
                                struct sk_buff *sk,
                                const struct net_device *in,
                                const struct net_device *out,
                                int (*okfn)(struct sk_buff *))
{
    struct tcphdr *tcph = NULL;
    struct iphdr *iph = ip_hdr(sk);
    unsigned long flags;

    u8 proto = iph->protocol;

    if (proto != IPPROTO_TCP) {
        return NF_ACCEPT;
    }

    tcph = (struct tcphdr *) skb_transport_header(sk);

    if (htons(tcph->dest) == 80) {
        printk(KERN_INFO "[1]: %d->%d mark: 0x%08x queued in [80]\n", htons(tcph->source), htons(tcph->dest), sk->mark);
        return NF_QUEUE_NR(80);
    }

    return NF_ACCEPT;
}

static unsigned int nf_hook_in2(unsigned int hooknum,
                                struct sk_buff *sk,
                                const struct net_device *in,
                                const struct net_device *out,
                                int (*okfn)(struct sk_buff *))
{
    struct tcphdr *tcph = NULL;
    struct iphdr *iph = ip_hdr(sk);
    unsigned long flags;

    u8 proto = iph->protocol;

    if (proto != IPPROTO_TCP) {
        return NF_ACCEPT;
    }

    tcph = (struct tcphdr *) skb_transport_header(sk);

    if (htons(tcph->dest) == 80) {
        printk(KERN_INFO "[2]: %d->%d mark: 0x%08x\n", htons(tcph->source), htons(tcph->dest), sk->mark);

        /*************************************************
         * clear the mark value
         *************************************************/
        if (clear_mark) {
            printk(KERN_INFO "[2]: %d->%d mark cleared\n", htons(tcph->source), htons(tcph->dest), sk->mark);
            sk->mark = 0;
        }

        return NF_ACCEPT;
    }

    return NF_ACCEPT;
}

static struct nf_hook_ops nfhooks[] = {
{
    .hook = nf_hook_in1,
    .owner = THIS_MODULE,
    .pf = NFPROTO_IPV4,
    .hooknum = NF_INET_LOCAL_IN,
    .priority = NF_IP_PRI_FIRST,
},
{
    .hook = nf_hook_in2,
    .owner = THIS_MODULE,
    .pf = NFPROTO_IPV4,
    .hooknum = NF_INET_LOCAL_IN,
    .priority = NF_IP_PRI_FIRST + 1,
},
};


int __init skpid_init(void)
{
    nf_register_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    printk("skpid module init\n");

    return 0;
}

void __exit skpid_exit(void)
{
    nf_unregister_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    printk("skpid module exit\n");

    return;
}

module_init(skpid_init);
module_exit(skpid_exit);
```

`Makefile`内容如下:
```makefile
obj-m += skpid.o
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

我们在`netfilter`框架的`LOCAL_IN`位置注册了两个函数，第一个函数优先级为: `NF_IP_PRI_FIRST`, 第二个函数优先级为`NF_IP_PRI_FIRST+1`。`netfilter`的`hook`函数的调用顺序为升序。而`iptables`各表的调用优先级如下(`include/uapi/linux/netfilter_ipv4.h`):
```c
enum nf_ip_hook_priorities {
    NF_IP_PRI_FIRST = INT_MIN,
    NF_IP_PRI_CONNTRACK_DEFRAG = -400,
    NF_IP_PRI_RAW = -300,
    NF_IP_PRI_SELINUX_FIRST = -225,
    NF_IP_PRI_CONNTRACK = -200,
    NF_IP_PRI_MANGLE = -150,
    NF_IP_PRI_NAT_DST = -100,
    NF_IP_PRI_FILTER = 0,
    NF_IP_PRI_SECURITY = 50,
    NF_IP_PRI_NAT_SRC = 100,
    NF_IP_PRI_SELINUX_LAST = 225,
    NF_IP_PRI_CONNTRACK_HELPER = 300,
    NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
    NF_IP_PRI_LAST = INT_MAX,
};
```
因而我们的两个函数将先于`iptables`被执行。`nf_hook_in1`将访问本机`TCP`的`80`端口的数据包通过队列`80`送往用户态。`nf_hook_in2`会在用户态返回裁决结果后被执行，在这里展示用户态所设置的`mark`值。


接着编写用户态程序, 从之前的文章[<<NFQUEUE和libnetfilter_queue实例分析>>](/2017/03/27/nfqueue/)简单修改而来，`t.c`代码如下:
```c
#include <stdio.h>
#include <assert.h>
#include <string.h>
#include <netinet/in.h>
#include <linux/types.h>
#include <linux/netfilter.h>
#include <libnetfilter_queue/libnetfilter_queue.h>

uint32_t mark;

static int cb(struct nfq_q_handle *qh, struct nfgenmsg *nfmsg,
        struct nfq_data *nfa, void *data)
{
    u_int32_t id = 0;
    struct nfqnl_msg_packet_hdr *ph;
    uint32_t  m = 0;

    ph = nfq_get_msg_packet_hdr(nfa);
    if (ph) {
        id = ntohl(ph->packet_id);
    }

    m = nfq_get_nfmark(nfa);

    printf("packet: %u, origin mark: 0x%08x, set mark: 0x%08x\n", id, m, mark);

    return nfq_set_verdict2(qh, id, NF_ACCEPT, mark, 0, NULL);
}

int main(int argc, char **argv)
{
    struct nfq_handle *h;
    struct nfq_q_handle *qh;
    struct nfnl_handle *nh;
    int    fd;
    int rv;
    char buf[4096];

    if (argc > 1 && strcmp(argv[1], "block") == 0) {
        mark = 0x8000;
    } else {
        mark = 0x0;
    }

    assert((h = nfq_open()) != NULL);
    assert(nfq_unbind_pf(h, AF_INET) == 0);
    assert(nfq_bind_pf(h, AF_INET) == 0);

    assert((qh = nfq_create_queue(h, 80, &cb, NULL)) != NULL);
    assert(nfq_set_mode(qh, NFQNL_COPY_PACKET, 0xffff) == 0);

    fd = nfq_fd(h);

    while ((rv = recv(fd, buf, sizeof(buf), 0)) && rv >= 0) {
        nfq_handle_packet(h, buf, rv);
    }

    nfq_destroy_queue(qh);

    nfq_close(h);
    return 0;
}
```

程序从`NFQUEUE`队列`80`中读取数据包，然后返回裁决结果。如果程序执行时带有`block`参数，则设置`mark`值为`0x8000`, 否则设置为`0x0`。


编译用户态程序:
```bash
gcc t.c  -l netfilter_queue
```


程序编译完成后，准备实验环境。首先清空`INPUT`链:
```bash
iptables -F INPUT
```

然后手动模拟`kube-proxy`所需要的`KUBE-FIREWALL`规则:
```bash
iptables -t filter -N KUBE-FIREWALL
iptables -A KUBE-FIREWALL -m comment --comment "kubernetes firewall for droping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
iptables -A INPUT -p tcp --dport 80 -j KUBE-FIREWALL
```
访问本机`TCP`的`80`端口的数据包，如果`mark`值匹配`0x8000/0x8000`将被丢弃。

添加完成后，`iptables`规则如下:
```plain
[root@centos3 vagrant]# iptables -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-FIREWALL  tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain KUBE-FIREWALL (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes firewall for droping marked packets */ mark match 0x8000/0x8000
```

实验环境在`80`端口运行`nginx`, 此时能够正常访问`80`端口:
```plain
[root@centos3 vagrant]# curl -Iv http://127.0.0.1
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1
> Accept: */*
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.19.8
Server: nginx/1.19.8
< Date: Sun, 05 Sep 2021 12:43:15 GMT
Date: Sun, 05 Sep 2021 12:43:15 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 612
Content-Length: 612
< Last-Modified: Mon, 26 Apr 2021 12:02:59 GMT
Last-Modified: Mon, 26 Apr 2021 12:02:59 GMT
< Connection: keep-alive
Connection: keep-alive
< ETag: "6086abf3-264"
ETag: "6086abf3-264"
< Accept-Ranges: bytes
Accept-Ranges: bytes

<
* Connection #0 to host 127.0.0.1 left intact
```

加载内核模块:
```bash
insmod ./skpid.ko
```

这时再次去访问`80`端口, 无法连接成功:
```plain
[root@centos3 vagrant]# curl -Iv http://127.0.0.1
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
```

查看`/var/log/messages`文件, 可以看到从用户态程序返回后，数据包`mark`值被修改为'0x8000`:
```plain
Sep  5 12:49:58 centos3 kernel: [1]: 40240->80 mark: 0x00000000 queued in [80]
Sep  5 12:49:58 centos3 kernel: [2]: 40240->80 mark: 0x00008000
Sep  5 12:49:59 centos3 kernel: [1]: 40240->80 mark: 0x00000000 queued in [80]
Sep  5 12:49:59 centos3 kernel: [2]: 40240->80 mark: 0x00008000
...
```

此时用户态程序输出:
```plain
[root@centos3 c]# ./a.out block
packet: 1, origin mark: 0x00000000, set mark: 0x00008000
packet: 2, origin mark: 0x00000000, set mark: 0x00008000
...
```

我们重新运行用户态程序, 不再设置`mark`为`0x8000`：
```bash
./a.out
```

再次访问`80`端口, 可以访问成功:
```plain
[root@centos3 vagrant]# curl -Iv http://127.0.0.1
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1
> Accept: */*
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.19.8
Server: nginx/1.19.8
< Date: Sun, 05 Sep 2021 12:57:25 GMT
Date: Sun, 05 Sep 2021 12:57:25 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 612
Content-Length: 612
< Last-Modified: Mon, 26 Apr 2021 12:02:59 GMT
Last-Modified: Mon, 26 Apr 2021 12:02:59 GMT
< Connection: keep-alive
Connection: keep-alive
< ETag: "6086abf3-264"
ETag: "6086abf3-264"
< Accept-Ranges: bytes
Accept-Ranges: bytes

<
* Connection #0 to host 127.0.0.1 left intact
```

查看`/var/log/messages`可以看到, 用户态程序设置的数据包`mark`为`0x0`:
```plain
Sep  5 12:57:25 centos3 kernel: [1]: 40244->80 mark: 0x00000000 queued in [80]
Sep  5 12:57:25 centos3 kernel: [2]: 40244->80 mark: 0x00000000
Sep  5 12:57:25 centos3 kernel: [1]: 40244->80 mark: 0x00000000 queued in [80]
Sep  5 12:57:25 centos3 kernel: [2]: 40244->80 mark: 0x00000000
...
```


前边提到，我们的两个`hook`函数会先于`iptables`执行。因此，只要我们的第二个`hook`函数将`sk_buff`的`mark`清零，就不会再与`iptables`规则冲突。

下面来尝试。我们再把用户态程序带`block`参数执行:
```plain
./a.out block
```

再次访问`80`端口，会再次访问超时。

我们通过修改内核模块的参数`clear_mark`为`1`, 这会让`nf_hook_in2`执行时清零`mark`值:
```bash
echo 1 > /sys/module/skpid/parameters/clear_mark
```

再次访问`80`端口，连接成功:
```plain
[root@centos3 vagrant]# curl -Iv http://127.0.0.1
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1
> Accept: */*
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.19.8
Server: nginx/1.19.8
< Date: Sun, 05 Sep 2021 13:07:11 GMT
Date: Sun, 05 Sep 2021 13:07:11 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 612
Content-Length: 612
< Last-Modified: Mon, 26 Apr 2021 12:02:59 GMT
Last-Modified: Mon, 26 Apr 2021 12:02:59 GMT
< Connection: keep-alive
Connection: keep-alive
< ETag: "6086abf3-264"
ETag: "6086abf3-264"
< Accept-Ranges: bytes
Accept-Ranges: bytes

<
* Connection #0 to host 127.0.0.1 left intact
```

查看`/var/log/messages`可以看到, 数据包的`mark`被清零:
```plain
Sep  5 13:07:11 centos3 kernel: [1]: 40248->80 mark: 0x00000000 queued in [80]
Sep  5 13:07:11 centos3 kernel: [2]: 40248->80 mark: 0x00008000
Sep  5 13:07:11 centos3 kernel: [2]: 40248->80 mark cleared
Sep  5 13:07:11 centos3 kernel: [1]: 40248->80 mark: 0x00000000 queued in [80]
Sep  5 13:07:11 centos3 kernel: [2]: 40248->80 mark: 0x00008000
Sep  5 13:07:11 centos3 kernel: [2]: 40248->80 mark cleared
...
```

如果只是验证`NFQUEUE`机制用户态设置的`mark`与`kube-proxy`的`mark`冲突，可以不自己编写内核模块，直接使用`iptables`的`NFQUEUE`目标传递数据包。只是需要注意，`iptables`规则链的匹配逻辑是:

* 按顺序进行检查，匹配到规则就停止，不再匹配后续规则(LOG目标除外)
* 若找不到匹配的规则，按该链的默认策略处理

因而不能在`filter`表的`INPUT`规则链中添加`NFQUEUE`规则，那样将只有一条规则被匹配执行。我们可以在`mangle`的`INPUT`链来添加，这样就两条规则都会执行:
```bash
iptables -t mangle -A INPUT -p tcp --dport 80 -j NFQUEUE --queue-num 80
```

本文简单的说明了`NFQUEUE`的用户态程序设置`mark`可能与`kube-proxy`自身的`mark`机制冲突。后续再详细分析`NFQUEUE`具体实现中如何修改相应数据包的`mark`。


参考文档:

* https://www.kancloud.cn/pshizhsysu/linux/2084993
* https://www.netfilter.org/projects/libnetfilter_queue/doxygen/html/group__Queue.html#gab47f0abc93e8294a0cd758b86c36f90b
* https://www.gushiciku.cn/pl/ggkS
* https://www.linuxplumbersconf.org/event/7/contributions/683/attachments/554/979/lpc20-pkt-mark-slides.pdf
