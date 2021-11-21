---
title: Linux内核TCP网络连接关联进程
date: 2021-07-21 19:02:48
tags:
- Network
- Kernel
categories: Network
description:
---

在一些服务器安全场景中，需要通过网络连接关联到相关进程。例如，在安全溯源场景中，通过威胁情报可以判断某台主机上存在恶意连接，这时就需要追查这些恶意连接是由哪个进程以及哪个可执行文件来发起的。又或者，在微隔离场景中，我们不仅仅需要知道`IP:Port`与`IP:Port`之间的访问关系，我们还需要额外增加进程级别的信息，也就是哪个进程通过`IP:Port`在访问`IP:Port`的哪个进程。

要解决这种网络连接与进程关联的问题，在用户态的可行办法主要是通过读取`/proc/net/tcp`以及`/proc/[pid]/fd`这两种文件来构建相应的映射结构。

通过读取文件`/proc/net/tcp`可获取系统的TCP连接信息:
```plain
[root@centos3 tcpconn]# cat /proc/net/tcp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
   0: 00000000:006F 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 13841 1 ffff9fc7da7c8000 100 0 0 10 0
   1: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 16863 1 ffff9fc7da7c87c0 100 0 0 10 0
   2: 0100007F:0019 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 17951 1 ffff9fc7da7c9f00 100 0 0 10 0
   3: 0F02000A:0016 0202000A:F3FE 01 00000000:00000000 02:000AF352 00000000     0        0 23961 4 ffff9fc7da7c8f80 20 4 25 10 -1
```

从其中可获取TCP连接四元组及对应socket的`inode`号。

而从`/proc/[pid]/fd`中可以获取进程所有的文件描述符:
```plain
[root@centos3 tcpconn]# ls -l /proc/823/fd
total 0
lr-x------. 1 root root 64 Jul 24 15:06 0 -> /dev/null
lrwx------. 1 root root 64 Jul 24 15:06 1 -> socket:[16389]
lrwx------. 1 root root 64 Jul 24 15:06 2 -> socket:[16389]
lrwx------. 1 root root 64 Jul 24 15:06 3 -> socket:[16863]
lrwx------. 1 root root 64 Jul 24 15:06 4 -> socket:[16939]
```

其中也可以获取相应的`inode`号.

这样，我们就可以从`/proc/net/tcp`建立起`网络连接五元组->inode`的映射, 再从`/proc/pid/fd`建立起连接`inode->进程`的映射。从而实现网络连接关联到相应进程。

但这种方式整个映射关系的建立依赖周期性读取两种proc文件，缺乏实时性，对于瞬时连接相应的数据可以无法实时获取到，从而无法关联到进程。

<!--more-->

为了实时获取网络和进程的关联情况，我们可以通过内核模块来实现。我们可以在TCP协议`socket`的`connect()`调用时，直接获取相应的进程信息，建立相应的关联关系，而在`close()`调时，再做相应的资源回收。

其中存在几个技术要点:
1. TCP协议`socket`相关的操作由全局结构变量`inet_stream_ops`来定义。我们可以替换该结构体的相应成员变量来实现我们的逻辑。
```
const struct proto_ops inet_stream_ops = {
	.family		   = PF_INET,
	.owner		   = THIS_MODULE,
	.release	   = inet_release,
	.bind		   = inet_bind,
	.connect	   = inet_stream_connect,
	.socketpair	   = sock_no_socketpair,
	.accept		   = inet_accept,
	.getname	   = inet_getname,
	.poll		   = tcp_poll,
	.ioctl		   = inet_ioctl,
	.listen		   = inet_listen,
	.shutdown	   = inet_shutdown,
	.setsockopt	   = sock_common_setsockopt,
	.getsockopt	   = sock_common_getsockopt,
	.sendmsg	   = inet_sendmsg,
	.recvmsg	   = inet_recvmsg,
	.mmap		   = sock_no_mmap,
	.sendpage	   = inet_sendpage,
	.splice_read	   = tcp_splice_read,
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_sock_common_setsockopt,
	.compat_getsockopt = compat_sock_common_getsockopt,
	.compat_ioctl	   = inet_compat_ioctl,
#endif
};
```

2、在3.X以上的内核版本里，这个结构体默认是不允许我们修改的。我们需要设置虚拟地址对应页表项的读写属性来实现可修改。

3、直接从`struct socket`结构无法直接获取到相应的进程信息。但因为我们的逻辑是在系统调用上下文中，可以通过`current`宏来获取正在运行的进程信息，从而进一步可获取到PID以及相应的二进制文件等等。

简要的示例代码如下:
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>
#include <linux/err.h>
#include <linux/time.h>
#include <linux/skbuff.h>
#include <linux/sched.h>
#include <net/tcp.h>
#include <net/inet_common.h>
#include <linux/uaccess.h>
#include <linux/netdevice.h>
#include <net/net_namespace.h>
#include <linux/mm.h>
#include <linux/kallsyms.h>
#include <net/ipv6.h>
#include <net/transp_v6.h>

unsigned long sk_data_ready_addr = 0;

static int
inet_stream_connect_tcpconn(struct socket *sock, struct sockaddr *uaddr,
        int addr_len, int flags)
{
    int retval = 0;
    char *pathname, *p;
    struct mm_struct *mm;
    struct sockaddr_in *in;
    struct inet_sock *inet;

    wait_queue_head_t *q;
    wait_queue_t *curr, *next;
    struct task_struct  *t;

    retval = inet_stream_connect(sock, uaddr, addr_len, flags);

    printk(KERN_INFO "inet_stream_connect_tcpconn called\n");

    if (sock && sock->sk) {
        inet = inet_sk(sock->sk);

        in = (struct sockaddr_in *)uaddr;
        if (in && inet) {
            printk(KERN_INFO "CPU [%u] CONN: %08x:%d->%08x:%d\n",
                    smp_processor_id(),
                    ntohl(inet->inet_saddr),
                    ntohs(inet->inet_sport),
                    ntohl(in->sin_addr.s_addr),
                    ntohs(in->sin_port));
        }
    }

    mm = get_task_mm(current);
    if (!mm) {
        goto out;
    }

    down_read(&mm->mmap_sem);
    if (mm->exe_file) {
        pathname = kmalloc(PATH_MAX, GFP_ATOMIC);
        if (pathname) {
            p = d_path(&mm->exe_file->f_path, pathname, PATH_MAX);

            printk(KERN_INFO "CPU [%u], FILE: %s, COMM: %s, PID: %d\n",
                    smp_processor_id(),
                    p, current->comm, current->pid);

            kfree(pathname);
        }
    }

    up_read(&mm->mmap_sem);

    mmput(mm);

out:
    return retval;
}

static inline int
hook_tcpconn_functions(void)
{
    unsigned int level;
    pte_t *pte;

    struct proto_ops *inet_stream_ops_p =
            (struct proto_ops *)&inet_stream_ops;

    pte = lookup_address((unsigned long)inet_stream_ops_p, &level);
    if (pte == NULL) {
        return 1;
    }

    if (pte->pte & ~_PAGE_RW) {
        pte->pte |= _PAGE_RW;
    }

    inet_stream_ops_p->connect = inet_stream_connect_tcpconn;
    printk(KERN_INFO "CPU [%u] hooked inet_stream_connect <%p> --> <%p>\n",
        smp_processor_id(), inet_stream_connect, inet_stream_ops_p->connect);

    return 0;
}

static int
unhook_tcpconn_functions(void)
{
    unsigned int level;
    pte_t *pte;

    struct proto_ops *inet_stream_ops_p =
            (struct proto_ops *)&inet_stream_ops;

    inet_stream_ops_p->connect = inet_stream_connect;
    printk(KERN_INFO "CPU [%u] unhooked inet_stream_connect\n",
        smp_processor_id());

    pte = lookup_address((unsigned long)inet_stream_ops_p, &level);
    if (pte == NULL) {
        return 1;
    }

    pte->pte |= pte->pte & ~_PAGE_RW;

    return 0;
}

static int __init
tcpconn_init(void)
{
    printk(KERN_INFO "loading tcpconn\n");

    sk_data_ready_addr = kallsyms_lookup_name("sock_def_readable");
    printk(KERN_INFO "CPU [%u] sk_data_ready_addr = "
        "kallsyms_lookup_name(sock_def_readable) = %lu\n",
         smp_processor_id(), sk_data_ready_addr);
    if (0 == sk_data_ready_addr) {
        printk(KERN_INFO "cannot find sock_def_readable.\n");
        goto err;
    }

    hook_tcpconn_functions();

    printk(KERN_INFO "tcpconn loaded\n");
    return 0;

err:
    return 1;
}

static void __exit
tcpconn_exit(void)
{
    unhook_tcpconn_functions();
    synchronize_net();

    printk(KERN_INFO "tcpconn unloaded\n");
}

module_init(tcpconn_init);
module_exit(tcpconn_exit);
MODULE_LICENSE("GPL");
```

在我们的逻辑中，获取到相应的进程信息之后，可以参考内核协议栈的`tcp_hashinfo`结构，建立相应的哈希表结构存储进程信息。而相应哈希条目的老化可以通过hook `close()`调用来实现。业务层逻辑可能在连接关闭的一段时间内仍然需要相应的映射信息，我们可以不立即删除相应条目，可以在这个位置给相应哈希条目设置过期时间以保留一段时间。具体的代码不再详述。


参考链接:
* https://zhuanlan.zhihu.com/p/49981590
