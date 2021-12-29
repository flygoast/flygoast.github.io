---
title: NFQUEUE机制导致DNS请求5秒超时分析
date: 2021-12-29 15:14:56
tags:
- Kernel
- Nfqueue
categories: Kernel
---
在一台`CentOS 7.0`服务器(内核版本号:`3.10.0-123.el7.x86_64`)上安装我们的安全防护程序后，会出现`curl`访问网址超时5秒的情况。现象如下:
```bash
[root@localhost ~]# time curl -s www.baidu.com -o /dev/null

real	0m5.120s
user	0m0.002s
sys	0m0.009s
```

通过`strace`分析程序调用的过程:
```
strace -f -tt -o curl.strace curl -s www.baidu.com -o /dev/null
```
从`strace`输出可以看到, 第一次`curl`调用`sendmmsg`同时发送了两个`DNS`数据包，分别是`A`记录和`AAAA`记录请求，但是只收到了`A`记录响应包:
{% img /images/2021-12-29/1.png %}

然后等待5秒超时后，依次调用`sendto`和`recvfrom`串行处理两个`DNS`请求, 这次两个`DNS`响应全部收到后，继续向下执行:
{% img /images/2021-12-29/2.png %}


而从抓包结果分析，`tcpdump`只能看到第一次同时发送的两个`DNS`请求中的`A`记录请求，`AAAA`记录请求数据包被内核协议栈丢弃了:
```plain
17:04:24.772049 IP 10.10.10.89.57416 > 114.114.114.114.53: 37081+ A? www.baidu.com. (31)
17:04:24.773693 IP 114.114.114.114.53 > 10.10.10.89.57416: 37081 3/0/0 CNAME www.a.shifen.com., A 180.101.49.12, A 180.101.49.11 (90)
17:04:29.776474 IP 10.10.10.89.57416 > 114.114.114.114.53: 37081+ A? www.baidu.com. (31)
17:04:29.778694 IP 114.114.114.114.53 > 10.10.10.89.57416: 37081 3/0/0 CNAME www.a.shifen.com., A 180.101.49.11, A 180.101.49.12 (90)
17:04:29.778925 IP 10.10.10.89.57416 > 114.114.114.114.53: 42471+ AAAA? www.baidu.com. (31)
17:04:29.780523 IP 114.114.114.114.53 > 10.10.10.89.57416: 42471 1/1/0 CNAME www.a.shifen.com. (115)
```

<!--more-->

在`Google`上搜索`DNS 5秒`有非常多关于类似现象的文章介绍，但基本都是在`Kubernetes`环境中发生。我们的环境只是普通的`CentOS`环境，为何也会发生呢？

`weave`公司的文章[Racy conntrack and DNS lookup timeouts](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts)介绍了在较旧版本内核的`conntrack`模块中存在的BUG会导致UDP丢包。其中一种场景是当不同的线程通过相同的`socket`发送`UDP`数据包时，存在竞争条件两个数据包都会各自创建一个`conntrack`条目，但两个条目所包含的`tuple`信息是一致的，这种情况会导致丢包。

从现象看，我们的场景丢包根因应该也是由于`conntrack`模块的BUG导致丢包，但我们的这种场景并不存在多个线程同时使用相同`socket`进行发送。我们的防护逻辑是内核模块通过`NFQUEUE`机制将数据包送到用户态，由用户态对数据包进行过滤裁决是否允许放行。因而我们怀疑是由于`NFQUEUE`机制导致`conntrack`模块这个已知BUG的触发。

我们写一个简单的内核模块将`DNS`请求通过`NFQUEUE`送到用户态, 用户态程序直接放行, 这样来验证能否复现问题。
内核模块代码，`nfqdns.c`内容:
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/skbuff.h>
#include <linux/ip.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <net/udp.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("nfqdns");
MODULE_ALIAS("module nfqdns netfiler");

static int nfqueue_no = 0;
MODULE_PARM_DESC(queue, "nfquene number");
module_param(nfqueue_no, int, 0600);

static unsigned int nf_hook_out(const struct nf_hook_ops *ops,
                   struct sk_buff *sk,
                   const struct net_device *in,
                   const struct net_device *out,
                   const struct nf_hook_state *state)
{
    struct udphdr *udph = NULL;
    struct iphdr *iph = ip_hdr(sk);

    u8 proto = iph->protocol;

    if (proto != IPPROTO_UDP) {
        return NF_ACCEPT;
    }

    udph = (struct udphdr *) skb_transport_header(sk);

    if (htons(udph->dest) == 53) {
        printk(KERN_INFO "[nfqdns]: %pI4:%d->%pI4:%d queued in [%d], skb: %p, ct: %p, tid: %d\n",
                &iph->saddr, htons(udph->source), &iph->daddr, htons(udph->dest), nfqueue_no,
                sk, sk->nfct, htons(*(unsigned short*)(udph + 1)));
        return NF_QUEUE_NR(nfqueue_no);
    }

    return NF_ACCEPT;
}

static struct nf_hook_ops nfhooks[] = {
{
    .hook = nf_hook_out,
    .owner = THIS_MODULE,
    .pf = NFPROTO_IPV4,
    .hooknum = NF_INET_POST_ROUTING,
    .priority = NF_IP_PRI_FIRST,
},
};


int __init nfqdns_init(void)
{
    nf_register_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    printk(KERN_INFO "nfqdns module init\n");

    return 0;
}

void __exit nfqdns_exit(void)
{
    nf_unregister_hooks(nfhooks, ARRAY_SIZE(nfhooks));

    printk(KERN_INFO "nfqdns module exit\n");

    return;
}

module_init(nfqdns_init);
module_exit(nfqdns_exit);
```

用户态程序`user.c`代码:
```c
#include <stdio.h>
#include <assert.h>
#include <string.h>
#include <netinet/in.h>
#include <linux/types.h>
#include <linux/netfilter.h>
#include <libnetfilter_queue/libnetfilter_queue.h>

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

    printf("packet: %u\n", id);

    return nfq_set_verdict(qh, id, NF_ACCEPT, 0, NULL);
}

int main(int argc, char **argv)
{
    struct nfq_handle *h;
    struct nfq_q_handle *qh;
    struct nfnl_handle *nh;
    int    fd;
    int rv;
    char buf[4096];

    assert((h = nfq_open()) != NULL);
    assert(nfq_unbind_pf(h, AF_INET) == 0);
    assert(nfq_bind_pf(h, AF_INET) == 0);

    assert((qh = nfq_create_queue(h, 0, &cb, NULL)) != NULL);
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

`Makefile`内容如下:
```makefile
obj-m += nfqdns.o
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
    gcc user.c  -l netfilter_queue -o user.out
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

因为BUG存在于`nf_conntrack`模块中，确保加载`nf_conntrack`和`nf_conntrack_ipv4`模块，然后加载我们的实验内核模块:
```bash
modprobe nf_conntrack nf_conntrack_ipv4
insmod ./nfqdns.ko
```

接着运行用户态程序:
```bash
./user.out
```

此时执行`curl`命令依然可以复现`DNS`超时现象，可以确定和`NFQUEUE`机制有关系。

从我们内核模块的日志输出可以看到在执行`nf_hook_out`时，`AAAA`记录的请求数据包还没有被丢弃，也可以看到第一次发送的两个`DNS`请求数据包的`conntrack`条目确实不同:
```plain
[467402.634931] [nfqdns]: 10.10.10.89:56578->114.114.114.114:53 queued in [0], skb: ffff880067d15d00, ct: ffff8800905ead68, tid: 8725
[467402.634958] [nfqdns]: 10.10.10.89:56578->114.114.114.114:53 queued in [0], skb: ffff880067d15c00, ct: ffff8800905ea750, tid: 5451
[467407.643516] [nfqdns]: 10.10.10.89:56578->114.114.114.114:53 queued in [0], skb: ffff8800b37f9f00, ct: ffff8800905ead68, tid: 8725
[467407.645559] [nfqdns]: 10.10.10.89:56578->114.114.114.114:53 queued in [0], skb: ffff8800b8fe3100, ct: ffff8800905ead68, tid: 5451
```

`conntrack`上述BUG丢弃数据包是发生在`conntrack`条目确认阶段的`__nf_conntrack_confirm`函数中。`conntrack`的原理和实现可以参考[这篇文章](https://arthurchiao.art/blog/conntrack-design-and-implementation-zh/), 本文不详述。

我们通过`systemtap`来验证丢包是否发生在这里。`systemtap`脚本`t.stp`内容如下:

```plain
%{
#include <net/udp.h>
%}

function read_tid: long (udphdr: long) %{
    struct udphdr *udph = (struct udphdr *)STAP_ARG_udphdr;
    unsigned short tid = htons(*(unsigned short *)(udph + 1));
    STAP_RETVALUE = (long) tid;
%}

probe module("nf_conntrack").function("__nf_conntrack_confirm") {
    iphdr = __get_skb_iphdr($skb)
    saddr = format_ipaddr(__ip_skb_saddr(iphdr), %{ AF_INET %})
    daddr = format_ipaddr(__ip_skb_daddr(iphdr), %{ AF_INET %})
    protocol = __ip_skb_proto(iphdr)

    udphdr = __get_skb_udphdr($skb)
    if (protocol == %{ IPPROTO_UDP %}) {
        dport = __tcp_skb_dport(udphdr)
        sport = __tcp_skb_sport(udphdr)
        if (dport == 53 || sport == 53) {
            printf("__nf_conntrack_confirm: %s:%d->%s:%d TID: %d, ct: %p, ctinfo: %d\n", saddr, sport, daddr, dport, read_tid(udphdr), $skb->nfct, $ctinfo)
        }
    }
}

probe module("nf_conntrack").function("__nf_conntrack_confirm").return {
    printf("__nf_conntrack_confirm: return %ld\n", $return)
}
```

从`systemtap`执行结果可以确定处理第二个`DNS`请求数据包时，函数`__nf_conntrack_confirm`函数返回了`0(NF_DROP)`:
```bash
[root@localhost fg]# stap -g t.stp
__nf_conntrack_confirm: 10.10.10.89:58305->114.114.114.114:53 TID: 16998, ct: 0xffff8800905eb860, ctinfo: 2
__nf_conntrack_confirm: return 1
__nf_conntrack_confirm: 10.10.10.89:58305->114.114.114.114:53 TID: 568, ct: 0xffff8800905eb248, ctinfo: 2
__nf_conntrack_confirm: return 0
```

而查看`3.10.0-123.el7.x86_64`版本的`__nf_conntrack_confirm(net/netfilter/nf_conntrack_core.c`)源码如下:
```c
/* Confirm a connection given skb; places it in hash table */
int
__nf_conntrack_confirm(struct sk_buff *skb)
{
        unsigned int hash, repl_hash;
        struct nf_conntrack_tuple_hash *h;
        struct nf_conn *ct;
        struct nf_conn_help *help;
        struct nf_conn_tstamp *tstamp;
        struct hlist_nulls_node *n;
        enum ip_conntrack_info ctinfo;
        struct net *net;
        u16 zone;

        ct = nf_ct_get(skb, &ctinfo);
        net = nf_ct_net(ct);

        /* ipt_REJECT uses nf_conntrack_attach to attach related
           ICMP/TCP RST packets in other direction.  Actual packet
           which created connection will be IP_CT_NEW or for an
           expected connection, IP_CT_RELATED. */
        if (CTINFO2DIR(ctinfo) != IP_CT_DIR_ORIGINAL)
                return NF_ACCEPT;

        zone = nf_ct_zone(ct);
        /* reuse the hash saved before */
        hash = *(unsigned long *)&ct->tuplehash[IP_CT_DIR_REPLY].hnnode.pprev;
        hash = hash_bucket(hash, net);
        repl_hash = hash_conntrack(net, zone,
                                   &ct->tuplehash[IP_CT_DIR_REPLY].tuple);

        /* We're not in hash table, and we refuse to set up related
           connections for unconfirmed conns.  But packet copies and
           REJECT will give spurious warnings here. */
        /* NF_CT_ASSERT(atomic_read(&ct->ct_general.use) == 1); */

        /* No external references means no one else could have
           confirmed us. */
        NF_CT_ASSERT(!nf_ct_is_confirmed(ct));
        pr_debug("Confirming conntrack %p\n", ct);

        spin_lock_bh(&nf_conntrack_lock);

        /* We have to check the DYING flag inside the lock to prevent
           a race against nf_ct_get_next_corpse() possibly called from
           user context, else we insert an already 'dead' hash, blocking
           further use of that particular connection -JM */

        if (unlikely(nf_ct_is_dying(ct))) {
                spin_unlock_bh(&nf_conntrack_lock);
                return NF_ACCEPT;
        }

        /* See if there's one in the list already, including reverse:
           NAT could have grabbed it without realizing, since we're
           not in the hash.  If there is, we lost race. */
        hlist_nulls_for_each_entry(h, n, &net->ct.hash[hash], hnnode)
                if (nf_ct_tuple_equal(&ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple,
                                      &h->tuple) &&
                    zone == nf_ct_zone(nf_ct_tuplehash_to_ctrack(h)))
                        goto out;
        hlist_nulls_for_each_entry(h, n, &net->ct.hash[repl_hash], hnnode)
                if (nf_ct_tuple_equal(&ct->tuplehash[IP_CT_DIR_REPLY].tuple,
                                      &h->tuple) &&
                    zone == nf_ct_zone(nf_ct_tuplehash_to_ctrack(h)))
                        goto out;

        /* Remove from unconfirmed list */
        hlist_nulls_del_rcu(&ct->tuplehash[IP_CT_DIR_ORIGINAL].hnnode);

        /* Timer relative to confirmation time, not original
           setting time, otherwise we'd get timer wrap in
           weird delay cases. */
        ct->timeout.expires += jiffies;
        add_timer(&ct->timeout);
        atomic_inc(&ct->ct_general.use);
        ct->status |= IPS_CONFIRMED;

        /* set conntrack timestamp, if enabled. */
        tstamp = nf_conn_tstamp_find(ct);
        if (tstamp) {
                if (skb->tstamp.tv64 == 0)
                        __net_timestamp(skb);

                tstamp->start = ktime_to_ns(skb->tstamp);
        }
        /* Since the lookup is lockless, hash insertion must be done after
         * starting the timer and setting the CONFIRMED bit. The RCU barriers
         * guarantee that no other CPU can find the conntrack before the above
         * stores are visible.
         */
        __nf_conntrack_hash_insert(ct, hash, repl_hash);
        NF_CT_STAT_INC(net, insert);
        spin_unlock_bh(&nf_conntrack_lock);

        help = nfct_help(ct);
        if (help && help->helper)
                nf_conntrack_event_cache(IPCT_HELPER, ct);

        nf_conntrack_event_cache(master_ct(ct) ?
                                 IPCT_RELATED : IPCT_NEW, ct);
        return NF_ACCEPT;

out:
        NF_CT_STAT_INC(net, insert_failed);
        spin_unlock_bh(&nf_conntrack_lock);
        return NF_DROP;
}
```
从代码可以看到，在从`conntrack`表中找到`tuple`信息一致的条目时，会跳转到`out`标签处，返回`NF_DROP`将数据包丢弃。

在更高版本的`CentOS`内核中修复了这个问题, `CentOS7.8`(内核版本:`3.10.0-1127.el7.x86_64`)的相同位置增加了冲突处理的逻辑来修复这个问题:
```c
out:
    nf_ct_add_to_dying_list(ct);
    ret = nf_ct_resolve_clash(net, skb, ctinfo, h);
dying:
    nf_conntrack_double_unlock(hash, reply_hash);
    NF_CT_STAT_INC(net, insert_failed);
    local_bh_enable();
    return ret;
```

从源码中我们也可以看到当出现这个问题时`conntrack`模块的`insert_failed`统计值会增加。我们可以通过`conntrack -S`命令来查看统计值。通过验证可以看到发生该现象时`insert_failed`统计值确实增加了:
{% img /images/2021-12-29/3.png %}

那么为什么`NFQUEUE`机制会触发该BUG呢？
从`curl`的`strace`输出可以看到，`curl`调用`sendmmsg`同时发送`A`和`AAAA`两个`DNS`请求。
系统调用`sendmmsg`的实现是`__sys_sendmmsg(net/socket.c)`, 它会循环调用`___sys_sendmsg`来发送两个数据包:
```c
        while (datagrams < vlen) {
                if (MSG_CMSG_COMPAT & flags) {
                        err = ___sys_sendmsg(sock, (struct msghdr __user *)compat_entry,
                                             &msg_sys, flags, &used_address);
                        if (err < 0)
                                break;
                        err = __put_user(err, &compat_entry->msg_len);
                        ++compat_entry;
                } else {
                        err = ___sys_sendmsg(sock,
                                             (struct msghdr __user *)entry,
                                             &msg_sys, flags, &used_address);
                        if (err < 0)
                                break;
                        err = put_user(err, &entry->msg_len);
                        ++entry;
                }

                if (err)
                        break;
                ++datagrams;
        }
```
具体网络发包过程可参考[这篇文章](https://zhuanlan.zhihu.com/p/373060740)。本文不详述。`___sys_sendmsg`函数会一直调用到协议栈的`ip_output(net/ipv4/ip_output.c)`函数:
```c
int ip_output(struct sk_buff *skb)
{
        struct net_device *dev = skb_dst(skb)->dev;

        IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUT, skb->len);

        skb->dev = dev;
        skb->protocol = htons(ETH_P_IP);

        return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL, dev,
                            ip_finish_output,
                            !(IPCB(skb)->flags & IPSKB_REROUTED));
}
```

`conntrack`实现中，外发连接的`conntrack`条目的创建是在`LOCAL_OUT`阶段，而条目确认也就是真正插入`conntrack`表是在`POST_ROUTING`阶段。这里的`NF_HOOK_COND`会迭代调用`netfilter`框架中注册的一系列`hook`函数。`nf_conntrack_ipv4`模块在该阶段注册了`ipv4_confirm(net/ipv4/netfilter/nf_conntrack_l3proto_ipv4.c)`函数，并且优先级为最低，最后才会执行。当没有加载我们的内核模块时，`NF_HOOK_COND`会一直执行到`ipv4_confirm`, 最终调用到`__nf_conntrack_confirm`函数，将`conntrack`条目插入`conntrack`表，之后层层返回。处理第二个数据包时，在`LOCAL_OUT`阶段处理时已经可以查到插入的`conntrack`条目，因而不会再给该数据包创建`conntrack`条目。而加载我们的内核模块后，我们在`POST_ROUTING`阶段注册的`nf_hook_out`函数优先级高于`nf_conntrack_ipv4`的`ipv4_confirm`函数，会先得到执行。我们返回`NF_QUEUE`之后，`NF_HOOK_COND`就会直接返回`0`。后续的函数执行需要内核收到用户态程序的`netlink`消息后再继续执行。因而`__nf_conntrack_confirm`这时并未得到执行。层层返回到`___sys_send_msg`之后，再发送第二个数据包。在`LOCAL_OUT`阶段由于在`conntrack`表中不能查找到相应的`conntrack`条目，所以会给该数据包再创建`conntrack`条目，最终触发BUG。

在存在该BUG的系统上可以通过在`/etc/resolv.conf`中添加`options single-request-reopen`来规避:

从`man 5 resolv.conf`可以看到相关说明:
```plain
single-request-reopen (since glibc 2.9)
     The resolver uses the same socket for the A and AAAA requests.  Some hardware mistakenly sends back only one reply.  When that happens the client system
     will  sit  and  wait for the second reply.  Turning this option on changes this behavior so that if two requests from the same port are not handled cor‐
     rectly it will close the  socket and open a new one before sending the second request.
```
这样就不会同时发送`A`和`AAAA`请求包，而是`A`请求收到回应包再发送`AAAA`请求，从而能在启用`NFQUEUE`机制的情况下规避该BUG。

后续有时间再从源码角度分析一下`conntrack`机制的实现。

参考:
* https://arthurchiao.art/blog/conntrack-design-and-implementation-zh/
* http://cxd2014.github.io/2017/08/15/connection-tracking-system/
* https://zhuanlan.zhihu.com/p/373060740
* https://www.codedump.info/post/20200128-systemtap-by-example/
