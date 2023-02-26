---
title: __pv_queued_spin_lock_slowpath崩溃分析
date: 2023-02-26 13:00:40
tags:
- Network
- Kernel
categories: Kernel
---
最近遇到一个`CentOS8`环境上的内核崩溃问题,内核版本号为`4.18.0-305.3.1.el8.x86_64`，崩溃堆栈为:
```plain
crash> bt
PID: 2310003  TASK: ffff99f4ee683e80  CPU: 1   COMMAND: "Verdict2"
 #0 [ffffb71241e375e8] machine_kexec at ffffffffbc66156e
 #1 [ffffb71241e37640] __crash_kexec at ffffffffbc78f99d
 #2 [ffffb71241e37708] crash_kexec at ffffffffbc79088d
 #3 [ffffb71241e37720] oops_end at ffffffffbc62434d
 #4 [ffffb71241e37740] no_context at ffffffffbc67262f
 #5 [ffffb71241e37798] __bad_area_nosemaphore at ffffffffbc67298c
 #6 [ffffb71241e377e0] do_page_fault at ffffffffbc673267
 #7 [ffffb71241e37810] page_fault at ffffffffbd0010fe
    [exception RIP: __pv_queued_spin_lock_slowpath+410]
    RIP: ffffffffbc73cbda  RSP: ffffb71241e378c0  RFLAGS: 00010282
    RAX: 0000000000003ffe  RBX: ffff99f4a6ffc624  RCX: 0000000000000001
    RDX: 0000000000003fff  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: ffff99f576e6ac00   R8: 0000000000000000   R9: ffff99f56428e200
    R10: 0000000032950000  R11: 0000000000000002  R12: ffffffffbcaaa6d0
    R13: ffff99f576e6ac14  R14: 0000000000000001  R15: 0000000000080000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #8 [ffffb71241e378f8] queued_write_lock_slowpath at ffffffffbc73df3c
 #9 [ffffb71241e37910] bpf_sk_reuseport_detach at ffffffffbc842ff9
#10 [ffffb71241e37928] reuseport_detach_sock at ffffffffbcdc2c25
#11 [ffffb71241e37940] sk_destruct at ffffffffbcd7ac33
#12 [ffffb71241e37950] nf_queue_entry_release_refs at ffffffffbce1c1e4
#13 [ffffb71241e37960] nf_reinject at ffffffffbce1c52e
#14 [ffffb71241e37998] nfqnl_recv_verdict at ffffffffc095a81f [nfnetlink_queue]
#15 [ffffb71241e37a10] nfnetlink_rcv_msg at ffffffffc09552be [nfnetlink]
#16 [ffffb71241e37b88] netlink_rcv_skb at ffffffffbce07a3c
#17 [ffffb71241e37bd8] nfnetlink_rcv at ffffffffc0955d08 [nfnetlink]
#18 [ffffb71241e37c18] netlink_unicast at ffffffffbce0725e
#19 [ffffb71241e37c58] netlink_sendmsg at ffffffffbce07524
#20 [ffffb71241e37cc8] sock_sendmsg at ffffffffbcd751fc
#21 [ffffb71241e37ce0] ____sys_sendmsg at ffffffffbcd7551b
#22 [ffffb71241e37d58] ___sys_sendmsg at ffffffffbcd76b9c
#23 [ffffb71241e37eb0] __sys_sendmsg at ffffffffbcd76c67
#24 [ffffb71241e37f38] do_syscall_64 at ffffffffbc60420b
#25 [ffffb71241e37f50] entry_SYSCALL_64_after_hwframe at ffffffffbd0000ad
```

<!--more-->

我们的场景是通过`NFQUEUE`机制将网络数据包上送到用户态，由用户态完成检测后再下发裁决放行或丢弃。从堆栈看崩溃就发生在用户态进程下发裁决的过程中，销毁`sock`结构的过程中调用自旋锁逻辑出现了崩溃。


反汇编函数`__pv_queued_spin_lock_slowpath`进行查看:
```plain
/usr/src/debug/kernel-4.18.0-305.3.1.el8_4/linux-4.18.0-305.3.1.el8.x86_64/kernel/locking/qspinlock.c: 139
0xffffffffbc73cbd2 <__pv_queued_spin_lock_slowpath+402>:        add    -0x42884760(,%rax,8),%r12
/usr/src/debug/kernel-4.18.0-305.3.1.el8_4/linux-4.18.0-305.3.1.el8.x86_64/./include/linux/compiler.h: 294
0xffffffffbc73cbda <__pv_queued_spin_lock_slowpath+410>:        mov    %rbp,(%r12)
/usr/src/debug/kernel-4.18.0-305.3.1.el8_4/linux-4.18.0-305.3.1.el8.x86_64/kernel/locking/qspinlock_paravirt.h: 301
0xffffffffbc73cbde <__pv_queued_spin_lock_slowpath+414>:        mov    $0x8000,%eax
0xffffffffbc73cbe3 <__pv_queued_spin_lock_slowpath+419>:        jmp    0xffffffffbc73cbfa <__pv_queued_spin_lock_slowpath+442>
```

结合源码可以确认崩溃在`WRITE_ONCE`的调用:
```c
        /*
         * if there was a previous node; link it and wait until reaching the
         * head of the waitqueue.
         */
        if (old & _Q_TAIL_MASK) {
                prev = decode_tail(old);

                /* Link @node into the waitqueue. */
                WRITE_ONCE(prev->next, node);

                pv_wait_node(node, prev);
                arch_mcs_spin_lock_contended(&node->locked);

                /*
                 * While waiting for the MCS lock, the next pointer may have
                 * been set by another lock waiter. We optimistically load
                 * the next pointer & prefetch the cacheline for writing
                 * to reduce latency in the upcoming MCS unlock operation.
                 */
                next = READ_ONCE(node->next);
                if (next)
                        prefetchw(next);
        }
```

崩溃时执行指令要写入的`r12`寄存器所指向的地址为非法值因而导致崩溃。
```plain
mov    %rbp,(%r12)
```

继续分析锁结构的内容。`__pv_queued_spin_lock_slowpath`开始执行时会将寄存器`rdi`值写入到`rbx`，后续`rbx`没有再被写入。因而此时，`rbx`的值`ffff99f4a6ffc624`为`struct qspinlock`参数的地址:
```plain
/usr/src/debug/kernel-4.18.0-305.3.1.el8_4/linux-4.18.0-305.3.1.el8.x86_64/kernel/locking/qspinlock.c: 325
0xffffffffbc73ca40 <__pv_queued_spin_lock_slowpath>:    nopl   0x0(%rax,%rax,1) [FTRACE NOP]
/usr/src/debug/kernel-4.18.0-305.3.1.el8_4/linux-4.18.0-305.3.1.el8.x86_64/kernel/locking/qspinlock.c: 409
0xffffffffbc73ca45 <__pv_queued_spin_lock_slowpath+5>:  push   %r15
0xffffffffbc73ca47 <__pv_queued_spin_lock_slowpath+7>:  mov    $0x2ac00,%rax
0xffffffffbc73ca4e <__pv_queued_spin_lock_slowpath+14>: push   %r14
0xffffffffbc73ca50 <__pv_queued_spin_lock_slowpath+16>: push   %r13
0xffffffffbc73ca52 <__pv_queued_spin_lock_slowpath+18>: push   %r12
0xffffffffbc73ca54 <__pv_queued_spin_lock_slowpath+20>: push   %rbp
0xffffffffbc73ca55 <__pv_queued_spin_lock_slowpath+21>: push   %rbx
0xffffffffbc73ca56 <__pv_queued_spin_lock_slowpath+22>: mov    %rdi,%rbx
0xffffffffbc73ca59 <__pv_queued_spin_lock_slowpath+25>: sub    $0x8,%rsp
```

查看锁结构数据，很明显数据不对:
```plain
crash> struct qspinlock ffff99f4a6ffc624
struct qspinlock {
  {
    val = {
      counter = 589823
    },
    {
      locked = 255 '\377',
      pending = 255 '\377'
    },
    {
      locked_pending = 65535,
      tail = 8
    }
  }
}
```

因而开始怀疑`sock`结构的内容被写脏了，继续分析`sock`结构:

通过偏移量可以计算出`sk`地址为`ffff99f4a6ffc3f0`:
```plain
crash> struct -xo sock.sk_callback_lock
struct sock {
  [0x230] rwlock_t sk_callback_lock;
}
crash> struct -xo rwlock_t
typedef struct {
  [0x0] arch_rwlock_t raw_lock;
} rwlock_t;
SIZE: 0x8
crash> struct -xo arch_rwlock_t
typedef struct qrwlock {
        union {
  [0x0]     atomic_t cnts;
            struct {
  [0x0]         u8 wlocked;
  [0x1]         u8 __lstate[3];
            };
        };
  [0x4] arch_spinlock_t wait_lock;
} arch_rwlock_t;
SIZE: 0x8
```

查看`sock`结构:
```plain
    skc_family = 0xa,
    skc_state = 0xc,
    skc_reuse = 0x6,
    skc_reuseport = 0x0,
    skc_ipv6only = 0x1,
    skc_net_refcnt = 0x0,
```

发现`sock`结构的`skc_reuseport`为`0`, 正常情况下并不应该调用到函数`reuseport_detach_sock`。

查看源码发现中判断是否调用`reuseport_detach_sock`是依赖字段`sk_reuseport_cb`:
```c
void sk_destruct(struct sock *sk)
{
        bool use_call_rcu = sock_flag(sk, SOCK_RCU_FREE);

        if (rcu_access_pointer(sk->sk_reuseport_cb)) {
                reuseport_detach_sock(sk);
                use_call_rcu = true;
        }

        if (use_call_rcu)
                call_rcu(&sk->sk_rcu, __sk_destruct);
        else
                __sk_destruct(&sk->sk_rcu);
}
```

查看字段`sk_reuseport_cb`发现指针明显不合法：
```c
crash> struct -x sock.sk_reuseport_cb  ffff99f4a6ffc3f0
  sk_reuseport_cb = 0x37cbbd77ffff0000
crash> struct -x sock_reuseport 0x37cbbd77ffff0000
struct: invalid kernel virtual address: 0x37cbbd77ffff0000
```

因而怀疑该`sock`结构并不是完整的`sock`结构。

继续查看`sock`结构状态，发现状态为`TCP_NEW_SYN_RECV`
```plain
    skc_state = 0xc,
```

```c
enum {
        TCP_ESTABLISHED = 1,
        TCP_SYN_SENT,
        TCP_SYN_RECV,
        TCP_FIN_WAIT1,
        TCP_FIN_WAIT2,
        TCP_TIME_WAIT,
        TCP_CLOSE,
        TCP_CLOSE_WAIT,
        TCP_LAST_ACK,
        TCP_LISTEN,
        TCP_CLOSING,    /* Now a valid state */
        TCP_NEW_SYN_RECV,

        TCP_MAX_STATES  /* Leave at the end! */
};
```

而状态为`TCP_NEW_SYN_RECV`状态时，此时的`sock`结构不是完整的`sock`结构，而是`tcp_request_sock`结构。

```plain
crash> struct -x tcp_request_sock
struct tcp_request_sock {
    struct inet_request_sock req;
    const struct tcp_request_sock_ops *af_specific;
    u64 snt_synack;
    bool tfo_listener;
    bool is_mptcp;
    bool drop_req;
    u32 txhash;
    u32 rcv_isn;
    u32 snt_isn;
    u32 ts_off;
    u32 last_oow_ack_time;
    u32 rcv_nxt;
}
SIZE: 0x148
crash> struct -xo sock.sk_callback_lock
struct sock {
  [0x230] rwlock_t sk_callback_lock;
}
```

`tcp_request_sock`结构的大小为`0x148`, 而`sk_callback_lock`的偏移为`0x230`， 因而会越界读取到脏数据，因而调用到`reuseport_detach_sock`路径。

继续从堆栈向上溯源查看`nf_queue_entry_release_refs`代码:
```c
static void nf_queue_entry_release_refs(struct nf_queue_entry *entry)
{
        struct nf_hook_state *state = &entry->state;

        /* Release those devices we held, or Alexey will kill me. */
        if (state->in)
                dev_put(state->in);
        if (state->out)
                dev_put(state->out);
        if (state->sk)
                sock_put(state->sk);

#if IS_ENABLED(CONFIG_BRIDGE_NETFILTER)
        if (entry->physin)
                dev_put(entry->physin);
        if (entry->physout)
                dev_put(entry->physout);
#endif
}
```

发现`nf_queue_entry_release_refs`是直接调用`sock_put`进行的引用计数释放，而未区分`sock`结构和`request_sock`结构，也就是在这里将`tcp_request_sock`当成`sock`结构来进行释放，最终导致崩溃。

内核官方已经修复了这个BUG:

* https://github.com/torvalds/linux/commit/747670fd9a2d1b7774030dba65ca022ba442ce71


如果要执行到本次崩溃的堆栈，需要`sk_refcnt`为`1`, 也就是`nfqueue`所持有的引用计数必须为最后一个:

```c
/* Ungrab socket and destroy it, if it was the last reference. */
static inline void sock_put(struct sock *sk)
{
        if (refcount_dec_and_test(&sk->sk_refcnt))
                sk_free(sk);
}
```


目前想到的场景可以有两种:
1. 内核回复`syn/ack`包后，在超时时间内没有收到`ack`包，于是重传`syn/ack`包，这个数据包被`NFQUEUE`送到用户态，`NFQUEUE`持有一个引用计数。而这时`ack`包到达，完成三次握手，用户态程序调用了`accept()`, 此时，`request_sock`从`listener`的队列中移除，只剩下`NFQUEUE`所持有的引用计数。
2. `NFQUEUE`持有`request_sock`的引用计数期间，`syn/ack`包的所有重传都已超时，这时`NFQUEUE`所持有的引用计数就是`request_sock`的唯一引用计数了。`SYN/ACK`重传参数在`CentOS`的默认值为`5`, 全部重传完成约`64`秒, 因而在实际场景下不太会出现，但可以将该值设置为`0`来构造该场景。
```c
[root@centos8-2 linux-4.18.0-305.3.1.el8.x86_64]# sysctl -a |grep synack
net.ipv4.tcp_synack_retries = 5
```

本文分析的崩溃就是第一种场景, 可以看到`tcp_request_sock`结构的`num_retrans`为`1`，表示重传了一次：
```plain
struct -x tcp_request_sock  ffff99f4a6ffc3f0
...
      dl_next = 0xffff99f523345a40,
      mss = 0x58c,
      num_retrans = 0x1,
      syncookie = 0x0,
      num_timeout = 0x1,
      ts_recent = 0x29596c,

      ...

      rsk_ops = 0xffffffffbde2a6e0,
      sk = 0xffff99f4fb463d80,
      saved_syn = 0x0,
      secid = 0x6806fc76,
      peer_secid = 0x1d810b44
...
```
`tcp_request_sock`结构中的`sk`字段也指向了新建的完整`sock`。


此外，从源码中看到，在将`tcp_request_sock`结构当作`sock`结构来释放的过程中，即使从`sock_put`调用到了`sk_free`, 如果`sk_wmem_alloc`值不为`1`也不会调用`__sk_free`, 只是将`sk_wmem_alloc`所对应的内存值减`1`：
```c
void sk_free(struct sock *sk)
{
        /*
         * We subtract one from sk_wmem_alloc and can know if
         * some packets are still in some tx queue.
         * If not null, sock_wfree() will call __sk_free(sk) later
         */
        if (refcount_dec_and_test(&sk->sk_wmem_alloc))
                __sk_free(sk);
}
```

而`sk_wmem_alloc`的偏移为`0x144`, 而`tcp_request_sock`的大小为`0x148`, 所修改的位置恰好是最后对齐的四个无用字节，因而没有其他副作用，只是单纯的造成该`tcp_request_sock`结构内存泄漏。

```plain
crash> struct -xo sock.sk_wmem_alloc
struct sock {
  [0x144] refcount_t sk_wmem_alloc;
}
crash> struct -xo tcp_request_sock
struct tcp_request_sock {
    [0x0] struct inet_request_sock req;
  [0x118] const struct tcp_request_sock_ops *af_specific;
  [0x120] u64 snt_synack;
  [0x128] bool tfo_listener;
  [0x129] bool is_mptcp;
  [0x12a] bool drop_req;
  [0x12c] u32 txhash;
  [0x130] u32 rcv_isn;
  [0x134] u32 snt_isn;
  [0x138] u32 ts_off;
  [0x13c] u32 last_oow_ack_time;
  [0x140] u32 rcv_nxt;
}
SIZE: 0x148
```

`CentOS8-stream`内核版本`4.18.0-373.el8.x86_64`之后修复了该BUG。分析清楚了具体的触发逻辑后，如果不能升级内核也可以通过`hook` `nf_reinject`函数来进行规避。

参考:
* http://lihanlu.cn/new-qspinlock/
* https://github.com/torvalds/linux/commit/747670fd9a2d1b7774030dba65ca022ba442ce71
* https://nan01ab.github.io/2019/10/Linux-NetStack.html
* https://www.cnblogs.com/Alexkk/p/12101950.html
* https://blog.csdn.net/sinat_20184565/article/details/88723621
* https://www.cnblogs.com/xingruizhi/p/14331785.html
* https://blog.csdn.net/u011130578/article/details/44954891
* https://wangquan.blog.csdn.net/article/details/108908194
* https://gohalo.me/post/network-synack-queue.html
* https://zhuanlan.zhihu.com/p/25313903
