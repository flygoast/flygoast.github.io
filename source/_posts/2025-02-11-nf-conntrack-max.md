---
title: 关于参数net.netfilter.nf_conntrack_max
date: 2025-02-11 15:27:44
tags:
- Network
- Kernel
- Netfilter
categories: Kernel
---
Linux内核中`conntrack`模块使用哈希表来存储连接跟踪条目，当哈希表条目达到上限时，系统会将新分配`conntrack`条目的数据包`DROP`掉，从而导致网络受到影响。此时，日志中会记录:
```plain
nf_conntrack: table full, dropping packet
```

哈希表条目上限由参数`net.netfilter.nf_conntrack_max`设置。

网上文章对这个问题的解决方法往往是调大该参数。但在涉及多个`network namespace`的场景下，不能简单的这样做，还是要根据自身场景分析清楚具体原因。

根据`CentOS7 3.10.0-957`版本内核源码，实际上每个`network namespace`的`conntrack`哈希表是独立的。在表示`network namespace`的结构体`net`中的成员`ct`表示`conntrack`相关信息:
```c
struct net {
    ...
    struct netns_ct     ct;
    ...
}
```
`netns_ct`结构中保存有独立的哈希表相关信息:
```c
struct netns_ct {
    atomic_t        count;
    ...
    unsigned int        htable_size;
    RH_KABI_DEPRECATE(seqcount_t, generation)
    struct kmem_cache   *nf_conntrack_cachep;
    struct hlist_nulls_head *hash;
    ...
}
```

<!--more-->

`nf_conntrack`模块的加载函数`nf_conntrack_standalone_init`注册了`pernet`操作:
```c
static struct pernet_operations nf_conntrack_net_ops = {
    .init       = nf_conntrack_pernet_init,
    .exit_batch = nf_conntrack_pernet_exit,
};
```

在每个`network namespace`创建时会执行函数`nf_conntrack_pernet_init`, 该函数中会调用函数`nf_conntrack_init_net`创建该`namespace`独有的`conntrack`表:
```c
    net->ct.slabname = kasprintf(GFP_KERNEL, "nf_conntrack_%p", net);
    if (!net->ct.slabname)
        goto err_slabname;

    net->ct.nf_conntrack_cachep = kmem_cache_create(net->ct.slabname,
                            sizeof(struct nf_conn), 0,
                            SLAB_DESTROY_BY_RCU, NULL);
    if (!net->ct.nf_conntrack_cachep) {
        printk(KERN_ERR "Unable to create nf_conn slab cache\n");
        goto err_cache;
    }

    net->ct.htable_size = nf_conntrack_htable_size;
    net->ct.hash = nf_ct_alloc_hashtable(&net->ct.htable_size, 1);
    if (!net->ct.hash) {
        printk(KERN_ERR "Unable to create nf_conntrack_hash\n");
        goto err_hash;
    }
```

在分配`conntrack`条目时，会调用函数`__nf_conntrack_alloc`函数:
```c
    if (nf_conntrack_max &&
        unlikely(atomic_read(&net->ct.count) > nf_conntrack_max)) {
        if (!early_drop(net, hash)) {
            atomic_dec(&net->ct.count);
            net_warn_ratelimited("nf_conntrack: table full, dropping packet\n");
            return ERR_PTR(-ENOMEM);
        }
    }
```

`nf_conntrack_max`参数对应`sysctl`参数:
```c
net.netfilter.nf_conntrack_max
```
这里会比较`namespace`独有的`conntrack`条目数和`nf_conntrack_max`参数值，如果达到了上限值，则会执行`early_drop`或导致数据包被丢弃。也就是说`nf_conntrack_max`表示的实际是每个`namespace`中`conntrack`条目的上限。

`nf_conntrack_max`本身是一个全局变量，并没有实现不同`namespace`之间的隔离:
```c
static struct ctl_table nf_ct_sysctl_table[] = {
    {
        .procname   = "nf_conntrack_max",
        .data       = &nf_conntrack_max,
        .maxlen     = sizeof(int),
        .mode       = 0644,
        .proc_handler   = proc_dointvec,
    },
    ...
```
也就是修改该参数会影响所有的`namespace`。

并且，`nf_conntrack_max`表示的是哈希表中的元素上限，它的默认取值是根据哈希表的`bucket`的平均长度来计算的:
```c
nf_conntrack_max = max_factor * nf_conntrack_htable_size;
```

当把`nf_conntrack_max`调大时，会导致哈希表`bucket`的平均长度变大，从而导致哈希表查找性能下降。因而还要考虑同步调整`bucket`的个数。

`bucket`个数变量`nf_conntrack_htable_size`也是全局的，并没有`namespace`隔离，在`CentOS7`上不能由`sysctl`变量修改，只能从内核模块参数修改:
```c
module_param_call(hashsize, nf_conntrack_set_hashsize, param_get_uint,
          &nf_conntrack_htable_size, 0600);
```

修改参数`nf_conntrack_htable_size`时，会调用函数`nf_conntrack_set_hashsize`:
```c
int nf_conntrack_set_hashsize(const char *val, struct kernel_param *kp)
{
    int i, bucket, rc;
    unsigned int hashsize, old_size;
    struct hlist_nulls_head *hash, *old_hash;
    struct nf_conntrack_tuple_hash *h;
    struct nf_conn *ct;

    if (current->nsproxy->net_ns != &init_net)
        return -EOPNOTSUPP;
```

这个函数会根据新的`bucket`个数创建新的哈希表，并将原来的`conntrack`条目迁移到新的哈希表。但是，实现上只处理`init_net`的哈希表。

其他`namespace`的`conntrack`表并没有进行调整。

因而，即使调整了`nf_conntrack_htable_size`, 其他的非`init_net`的`namespace`中的`conntrack`哈希表也不会调整。从而导致`bucket`的平均长度变大，性能下降。


再来看`CentOS8 4.18.0-348.7.1`版本内核的情况。

社区版本的这个commit:

* https://github.com/torvalds/linux/commit/56d52d4892d0e478a005b99ed10d0a7f488ea8c1

将系统上所有`namespace`的`conntrack`条目使用一个哈希表存储。`CentOS8`是包含这个提交。

这样能够解决上面说的非`init_net`中，修改`nf_conntrack_htable_size`参数而不重新构建哈希表的问题。但`__nf_conntrack_alloc`函数依然使用的是`namespace`的`conntrack`条目数与`nf_conntrack_max`进行比较进行上限判断:
```c
    if (nf_conntrack_max &&
        unlikely(atomic_read(&net->ct.count) > nf_conntrack_max)) {
        if (!early_drop(net, hash)) {
            if (!conntrack_gc_work.early_drop)
                conntrack_gc_work.early_drop = true;
            atomic_dec(&net->ct.count);
            net_warn_ratelimited("nf_conntrack: table full, dropping packet\n");
            return ERR_PTR(-ENOMEM);
        }
    }
```
如果系统上有多个`namespace`, 每个`namespace`中的条目都很多，但并未超过`nf_conntrack_max`值限制，但实际上`conntrack`哈希表的平均`bucket`长度则会与`namespace`正相关，会非常长，导致`conntrack`表性能下降。

对于我们已知可以跳过连接跳踪的场景，可以通过跳过没有必要的连接跟踪逻辑，从而避免`conntrack`表条目达到上限。

在连接跟踪入口函数`nf_conntrack_in`中会根据`sk_buff`结构来判断是否需要连接跟踪:
```c
    tmpl = nf_ct_get(skb, &ctinfo);
    if (tmpl || ctinfo == IP_CT_UNTRACKED) {
        /* Previously seen (loopback or untracked)?  Ignore. */
        if ((tmpl && !nf_ct_is_template(tmpl)) ||
             ctinfo == IP_CT_UNTRACKED)
            return NF_ACCEPT;
        skb->_nfct = 0;
    }
```

因而我们可以在`netfilter`框架中注册函数，在`nf_conntrack_in`执行前通过修改数据包跳过连接跟踪:
```c
nf_ct_set(skb, NULL, IP_CT_UNTRACKED);
```

参考:
* https://www.cnblogs.com/jianyungsun/p/12554455.html
* https://github.com/torvalds/linux/commit/56d52d4892d0e478a005b99ed10d0a7f488ea8c1
* https://www.haxi.cc/archives/nf_conntrack%E8%B0%83%E4%BC%98.html
