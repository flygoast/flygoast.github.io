---
title: netfilter连接跟踪模块扩展的相关问题
date: 2024-11-27 17:36:22
tags:
- Network
- Kernel
categories: Kernel
---
我们的网络防护功能是基于`netfilter`框架实现，依赖于`nf_conntrack`模块用于跟踪网络连接。在网络连接的维度，我们需要存储一些业务相关的数据。最简单直接的方法就是将这些内容存储在`nf_conn`结构中。

查看`nf_conn`结构，发现`nf_conn`结构中有一个指针`ext`可以支持扩展:

```C
struct nf_conn {
    /* Usage count in here is 1 for hash table, 1 per skb,
     * plus 1 for any connection(s) we are `master' for
     *
     * Hint, SKB address this struct and refcnt via skb->_nfct and
     * helpers nf_conntrack_get() and nf_conntrack_put().
     * Helper nf_ct_put() equals nf_conntrack_put() by dec refcnt,
     * beware nf_ct_get() is different and don't inc refcnt.
     */
    struct nf_conntrack ct_general;

    ......

    /* Extensions */
    struct nf_ct_ext *ext;

    /* Storage reserved for other modules, must be the last member */
    union nf_conntrack_proto proto;
};
```

`nf_ct_ext`结构:

```C
/* Extensions: optional stuff which isn't permanently in struct. */
struct nf_ct_ext {
    struct rcu_head rcu;
    u8 offset[NF_CT_EXT_NUM];
    u8 len;
    char data[0];
};
```

<!--more-->

然而，从代码中可以看到，该结构体只能支持已经明确的扩展功能，并不能让我们跟据我们的需求自行动态扩展，只能重新编译内核。

如果确定环境上不会使用某个扩展，确实不想重新编译内核，可以盗用这个扩展的ID，比如`NF_CT_EXT_ACCT`，但这样不能满足我们普适安装的需求。

这里感觉内核实现确实过于简单，并且这么多年来没有人去折腾它。这里完全可以实现成动态注册的机制，`nf_ct_extend`只保证调用已注册扩展的相关回调就可以。

我们无力改变内核现状，只能寻求其他解决方案。

既然无法直接将数据与`nf_conn`结构进行关联，我们可以采用间接关联，于是我们构建了一个哈希表，以`nf_conn`为`key`, 需要使用我们的扩展数据时，通过`nf_conn`从哈希表中查找。关联关系有了，要解决的问题就是存储扩展数据的内存的分配和释放时机。

分配时机比较简单，在哈希表中没有找到时，就可以分配并加入到哈希表。既然是扩展数据，释放时机最理想的就是跟随`nf_conn`结构的释放而释放。内核中`nf_conn`的释放恰好是通过指针`nf_ct_hook`来进行的：

```text-x-csrc
void nf_conntrack_destroy(struct nf_conntrack *nfct)
{
    struct nf_ct_hook *ct_hook;

    rcu_read_lock();
    ct_hook = rcu_dereference(nf_ct_hook);
    BUG_ON(ct_hook == NULL);
    ct_hook->destroy(nfct);
    rcu_read_unlock();
}
EXPORT_SYMBOL(nf_conntrack_destroy);
```

我们可以通过替换掉`nf_ct_hook`结构的`.destory`函数，先释放我们的扩展数据再调用原始的`.destroy`函数释放`nf_conn`结构。

这样就通过一个哈希表和替换释放函数的指针满足我们的需求。尽管不是非常完美，但这个机制一直运行良好。

然而最近我们在高一些的内核版本上，发现存在着内存泄漏。调研发现，`5.15.36`版本之后，内核修改了`nf_ct_put`的实现，它不再调用那个指针，而直接调用`nf_ct_destroy`函数：

```text-x-csrc
/* decrement reference count on a conntrack */
static inline void nf_ct_put(struct nf_conn *ct)
{
	if (ct && refcount_dec_and_test(&ct->ct_general.use))
		nf_ct_destroy(&ct->ct_general);
}
```

提交的commit是: https://github.com/torvalds/linux/commit/6ae7989c9af0d98ab64196f4f4c6f6499454bd23#diff-f09c467cde3ef955aab4ae4f70888393f1e72bbc481def21095127d858735236

这样，如果`nf_conn`结构是由`nf_ct_put`直接释放的情况下，我们的扩展内存就没有机会得到释放了。

为了解决这个问题，可以有两种方法：

*   不再替换`nf_ct_hook`结构的`.destroy`函数，而是直接替换`nf_ct_destory`函数
*   给我们的扩展数据添加超时过期机制，超过若干时间没有网络包触达的扩展内存就过期释放

两种方法都可以解决这个问题，甚至为了保险，防止有其他意外，两个方式都可以加上。

参考:

*   [https://blog.csdn.net/xiaoyu\_750516366/article/details/89281023](https://blog.csdn.net/xiaoyu_750516366/article/details/89281023)
*   [https://blog.51cto.com/dog250/1605198](https://blog.51cto.com/dog250/1605198)
*   [https://github.com/torvalds/linux/commit/6ae7989c9af0d98ab64196f4f4c6f6499454bd23#diff-f09c467cde3ef955aab4ae4f70888393f1e72bbc481def21095127d858735236](https://github.com/torvalds/linux/commit/6ae7989c9af0d98ab64196f4f4c6f6499454bd23#diff-f09c467cde3ef955aab4ae4f70888393f1e72bbc481def21095127d858735236)
