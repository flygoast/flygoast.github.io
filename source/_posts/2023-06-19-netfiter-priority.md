---
title: netfilter中相同优先级的HOOK函数的执行顺序
date: 2023-06-19 20:04:11
tags:
- Network
- Kernel
categories: Kernel
---
之前的两篇文章[<<nf_ct_deliver_cached_events崩溃分析>>](/2022/06/25/nf-ct-deliver-cached-events/)和[<<nf_ct_deliver_cached_events崩溃修复或规避方案>>](/2023/02/15/ct-fix/)介绍了`nf_conntrack`模块中的一个BUG的原因和规避方案。触发BUG的原因在于`NFQUEUE`操作位于`ipv4_conntrack_in`和`ipv4_confirm`两个函数之间，于是本可以无中断执行完成的两个函数之间出现了`CPU`调度，导致大量`conntrack entry`冲突。各`HOOK`函数执行顺序如图:

{% img /images/2023-06-19/1.png %}

<!--more-->

规避问题出现的思路就是将`NFQUEUE`操作不放在两个函数中间。之前文章中选定的思路是在我们的`hook`函数中提前调用`ipv4_confirm`。

最近，同事提出了另一个思路，就是将我们的`hook`函数的优先级也设置为`NF_IP_PRI_LAST`。因为`ipv4_confirm`的优先级`NF_IP_PRI_CONNTRACK_CONFIRM`是`INT_MAX`, 我们的`hook`函数优先级也设置成`INT_MAX`。因为我们的内核模块依赖`conntrack`模块，因而`conntrack`模块会先于我们的内核模块先注册，从而保证我们的`HOOK`函数最后执行。

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

但这引发我思考一个问题，相同优先级的`HOOK`函数的先后顺序能得到保证吗？

查看`CentOS7`的`3.10.0-862.el7.x86_64`版本内核上的`nf_register_hook`代码如下:
```c
int nf_register_hook(struct nf_hook_ops *reg)
{
        struct nf_hook_ops *elem;
        int err;

        err = mutex_lock_interruptible(&nf_hook_mutex);
        if (err < 0)
                return err;
        list_for_each_entry(elem, &nf_hooks[reg->pf][reg->hooknum], list) {
                if (reg->priority < elem->priority)
                        break;
        }
        list_add_rcu(&reg->list, elem->list.prev);
        mutex_unlock(&nf_hook_mutex);
#if defined(CONFIG_JUMP_LABEL)
        static_key_slow_inc(&nf_hooks_needed[reg->pf][reg->hooknum]);
#endif
        return 0;
}
EXPORT_SYMBOL(nf_register_hook);
```

可以看到这个版本上的排序比较条件使用的是:
```c
(reg->priority < elem->priority)
```

因而，后注册的相同优先级的`hook`函数的确会排在先注册的相同优先级的最后位置，可以保证相同优先级的`HOOK`函数先注册先执行。示意图如下:
{% img /images/2023-06-19/2.png %}

再来看`CentOS8`的`4.18`内核。函数`nf_register_net_hook`最终会调用到`nf_hook_entries_grow`函数进行排序:
```c
static struct nf_hook_entries *
nf_hook_entries_grow(const struct nf_hook_entries *old,
                     const struct nf_hook_ops *reg)
{
        unsigned int i, alloc_entries, nhooks, old_entries;
        struct nf_hook_ops **orig_ops = NULL;
        struct nf_hook_ops **new_ops;
        struct nf_hook_entries *new;
        bool inserted = false;

        alloc_entries = 1;
        old_entries = old ? old->num_hook_entries : 0;

        if (old) {
                orig_ops = nf_hook_entries_get_hook_ops(old);

                for (i = 0; i < old_entries; i++) {
                        if (orig_ops[i] != &dummy_ops)
                                alloc_entries++;
                }
        }

        if (alloc_entries > MAX_HOOK_COUNT)
                return ERR_PTR(-E2BIG);

        new = allocate_hook_entries_size(alloc_entries);
        if (!new)
                return ERR_PTR(-ENOMEM);

        new_ops = nf_hook_entries_get_hook_ops(new);

        i = 0;
        nhooks = 0;
        while (i < old_entries) {
                if (orig_ops[i] == &dummy_ops) {
                        ++i;
                        continue;
                }

                if (inserted || reg->priority > orig_ops[i]->priority) {
                        new_ops[nhooks] = (void *)orig_ops[i];
                        new->hooks[nhooks] = old->hooks[i];
                        i++;
                } else {
                        new_ops[nhooks] = (void *)reg;
                        new->hooks[nhooks].hook = reg->hook;
                        new->hooks[nhooks].priv = reg->priv;
                        inserted = true;
                }
                nhooks++;
        }

        if (!inserted) {
                new_ops[nhooks] = (void *)reg;
                new->hooks[nhooks].hook = reg->hook;
                new->hooks[nhooks].priv = reg->priv;
        }

        return new;
}
```

`4.18`版本上最终的`HOOK`函数存储结构做了修改，使用的排序条件也发生了变化:
```c
reg->priority > orig_ops[i]->priority
```

这与`3.10`版本上的判断条件是不同的，它会导致相同优先级的后注册`HOOK`函数排在相同优先级的`HOOK`函数的最前边位置。示意图如下:
{% img /images/2023-06-19/3.png %}

因而，相同优先级的`HOOK`函数先注册后执行。


可以使用如下的内核模块来进行测试:
```c
#define pr_fmt(fmt) "[%s]: " fmt, KBUILD_MODNAME

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>
#include <linux/init.h>
#include <linux/skbuff.h>
#include <linux/ip.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <net/tcp.h>


MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("nfq priority");


static unsigned int nf_hook1(void *priv,
                             struct sk_buff *skb,
                             const struct nf_hook_state *state)
{
    struct iphdr *iph = ip_hdr(skb);
    struct tcphdr *tcph;

    u8 proto = iph->protocol;

    if (proto != IPPROTO_TCP) {
        return NF_ACCEPT;
    }

    tcph = tcp_hdr(skb);
    if (ntohs(tcph->source) != 80) {
        return NF_ACCEPT;
    }

    pr_info("HOOK: 1, TCP  %d->%d\n", htons(tcph->source), htons(tcph->dest));

    return NF_ACCEPT;
}

static unsigned int nf_hook2(void *priv,
                             struct sk_buff *skb,
                             const struct nf_hook_state *state)
{
    struct iphdr *iph = ip_hdr(skb);
    struct tcphdr *tcph;

    u8 proto = iph->protocol;

    if (proto != IPPROTO_TCP) {
        return NF_ACCEPT;
    }

    tcph = tcp_hdr(skb);
    if (ntohs(tcph->source) != 80) {
        return NF_ACCEPT;
    }

    pr_info("HOOK: 2, TCP  %d->%d\n", htons(tcph->source), htons(tcph->dest));

    return NF_ACCEPT;
}

static struct nf_hook_ops nfhooks[] = {
    {
        .hook = nf_hook1,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_POST_ROUTING,
        .priority = NF_IP_PRI_CONNTRACK_CONFIRM,
    },
    {
        .hook = nf_hook2,
        .pf = NFPROTO_IPV4,
        .hooknum = NF_INET_POST_ROUTING,
        .priority = NF_IP_PRI_CONNTRACK_CONFIRM,
    },
};

int __init nfqprio_init(void)
{
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4,13,0)
    nf_register_net_hooks(&init_net, nfhooks, ARRAY_SIZE(nfhooks));
#else
    nf_register_hooks(nfhooks, ARRAY_SIZE(nfhooks));
#endif

    pr_info("module init\n");

    return 0;
}

void __exit nfqprio_exit(void)
{
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4,13,0)
    nf_unregister_net_hooks(&init_net, nfhooks, ARRAY_SIZE(nfhooks));
#else
    nf_unregister_hooks(nfhooks, ARRAY_SIZE(nfhooks));
#endif

    pr_info("module exit\n");

    return;
}

module_init(nfqprio_init);
module_exit(nfqprio_exit);
```


而在`CentOS7`上执行`curl`测试:
```bash
curl http://127.0.0.1
```

日志为:
```plain
[29588.265123] [nfqprio]: HOOK: 1, TCP  80->28512
[29588.265280] [nfqprio]: HOOK: 2, TCP  80->28512
```

而在`CentOS 8.2(4.18.0-193.el8.x86_64)`上执行后的日志为:
```plain
[8999405.679059] [nfqprio]: HOOK: 2, TCP  80->32794
[8999405.679635] [nfqprio]: HOOK: 1, TCP  80->32794
```


可以看到，在`CentOS7`的`3.10`版本内核上相同优先级的`HOOK`函数是先注册先执行，而在`CentOS8`的`4.18`内核上是先注册后执行。

如果只有一种内核环境，可以使用这个思路来进行规避。但对于我们的产品来说，从不同版本兼容性考虑，还是不采用这种思路，依然使用之前文章介绍的思路。
