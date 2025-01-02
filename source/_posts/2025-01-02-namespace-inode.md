---
title: 内核模块识别network namespace
date: 2025-01-02 18:39:42
tags:
- Kernel
- Network
categories: Kernel
---
业务场景中，需要创建指定的`network namespace`, 并且内核模块中的`netfilter`逻辑只应生效在该`network namespace`中。这就需要我们创建`network namespace`之后，将指定的`namespace`传递给内核模块。

用户态创建`network namespace`可以使用`ip`命令指定名称，如:

```text-plain
ip netns add ns1
```

但实际上在内核中，`network namespace`并不具备名称信息，名称信息只存在于用户态。可以参考`man ip-netns`:

```text-plain
       By convention a named network namespace is an object at /var/run/netns/NAME that can be opened. The file descriptor resulting
       from opening /var/run/netns/NAME refers to the specified network namespace. Holding that file descriptor open keeps the network
       namespace alive. The file descriptor can be used with the setns(2) system call to change the network namespace associated with a
       task.
```

<!--more-->

但`network namespace`在`/proc`文件系统中存在，因而可以使用文件的`inode`来标识。`inode`信息则可以通过`network namespace`中的进程从`/proc`获取到`namespace`的文件来识别。

下边来实验。
先在上边创建的`ns1`中执行一个`sleep 3600`：

```text-plain
ip netns exec ns1 sleep 3600
```

通过`pid`获取`network namespace`的`inode`:

```text-plain
[root@localhost ~]# ps aux |grep sleep
root     22371  0.0  0.0 108052   356 pts/2    S+   17:59   0:00 sleep 3600
root     22620  0.0  0.0   9092   680 pts/3    S+   18:01   0:00 grep --color=auto sleep
[root@localhost ~]# ls -l /proc/22371/ns/net
lrwxrwxrwx 1 root root 0 Jan  2 17:59 /proc/22371/ns/net -> net:[4026532631]
```

在`CentOS7`的内核源码中，`inode`信息保存在`struct net`结构的`proc_inum`字段中。后来迁移到了`struct ns_common`结构中:

*   [https://github.com/torvalds/linux/commit/435d5f4bb2ccba3b791d9ef61d2590e30b8e806e#diff-3d2e3d4731c5acd84fc41b7c480470d3ae2ca16b45525b0a30b29b9d209b7332R64](https://github.com/torvalds/linux/commit/435d5f4bb2ccba3b791d9ef61d2590e30b8e806e#diff-3d2e3d4731c5acd84fc41b7c480470d3ae2ca16b45525b0a30b29b9d209b7332R64)

不同版本的内核需要兼容处理一下。这里以`CentOS7`为实验环境，编写内核模块源码如下:

```text-plain
#define pr_fmt(fmt) "[%s]: " fmt, KBUILD_MODNAME

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/net.h>
#include <net/net_namespace.h>
#include <linux/proc_ns.h>
#include <linux/nsproxy.h>
#include <linux/dcache.h>
#include <linux/sched.h>


static unsigned int inode = 0;
module_param(inode, uint, 0400);

int __init netns_inode_init(void)
{
    struct net *net;

    pr_info("netns_inode module init\n");

    for_each_net(net) {
        pr_info("namespace: p: %px, inode: %u%s", net, net->proc_inum, (inode == net->proc_inum) ? " ***\n" : "\n");
    }

    return -1;
}

void __exit netns_inode_exit(void)
{
    pr_info("netns_inode module exit\n");
    return;
}

module_init(netns_inode_init);
module_exit(netns_inode_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("netns inode");
```

编译后执行内核模块，可以识别到传入的`inode`标识:

```text-plain
[root@localhost netns_inode]# insmod netns_inode.ko inode=4026532631
insmod: ERROR: could not insert module netns_inode.ko: Operation not permitted
[root@localhost netns_inode]# dmesg
[1216768.454246] [netns_inode]: netns_inode module init
[1216768.454250] [netns_inode]: namespace: p: ffffffffa3111640, inode: 4026531956
[1216768.454252] [netns_inode]: namespace: p: ffffa0c93f6b9480, inode: 4026532511
[1216768.454253] [netns_inode]: namespace: p: ffffa0c91c738000, inode: 4026532574
[1216768.454254] [netns_inode]: namespace: p: ffffa0c613f70000, inode: 4026532631 ***
```
