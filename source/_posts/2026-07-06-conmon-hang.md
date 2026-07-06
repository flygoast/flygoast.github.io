---
title: conmon 2.0.29 exec会话僵尸进程根因分析
date: 2026-07-06 14:42:41
tags:
- conmon
categories: MISC

---
## 问题现象

我们的程序通过 `podman exec` 在容器中周期性执行命令，偶尔会出现命令进程退出后成为僵尸进程`zombie`，一直不被回收。查看进程关系，僵尸进程的父进程为`conmon`。此时`conmon`进程阻塞在`poll()` 系统调用，无法回收已退出的命令进程。

```plain
[root@dev05 ~]# ps -ef |grep 1580889
root     1580889       1  0 Jul02 ?        00:00:00 /usr/bin/conmon --api-version 1 -c c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5 -u 98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd -r /usr/bin/runc -b /var/lib/containers/storage/overlay-containers/c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5/userdata/98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd -p /var/lib/containers/storage/overlay-containers/c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5/userdata/98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd/exec_pid -n tomcat --exit-dir /var/lib/containers/storage/overlay-containers/c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5/userdata/98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd/exit --full-attach -l none --log-level warning --runtime-arg --log-format=json --runtime-arg --log --runtime-arg=/var/lib/containers/storage/overlay-containers/c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5/userdata/98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd/oci-log -i -e --exec-attach --exec-process-spec /var/lib/containers/storage/overlay-containers/c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5/userdata/98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd/exec-process-454711737 --exit-command /usr/bin/podman --exit-command-arg --root --exit-command-arg /var/lib/containers/storage --exit-command-arg --runroot --exit-command-arg /run/containers/storage --exit-command-arg --log-level --exit-command-arg warning --exit-command-arg --cgroup-manager --exit-command-arg cgroupfs --exit-command-arg --tmpdir --exit-command-arg /run/libpod --exit-command-arg --runtime --exit-command-arg runc --exit-command-arg --storage-driver --exit-command-arg overlay --exit-command-arg --storage-opt --exit-command-arg overlay.mountopt=nodev,metacopy=on --exit-command-arg --events-backend --exit-command-arg file --exit-command-arg container --exit-command-arg cleanup --exit-command-arg --exec --exit-command-arg 98aaaceda2866ee23fc0a5e2e355c696bbb5d9810104331efdacaf31ea33cdcd --exit-command-arg c6b34b14a6622562eb4b076c021c52ba9f55746cd51849638126577328c798a5
root     1580901 1580889  0 Jul02 ?        00:00:00 [md5sum] <defunct>
[root@dev05 tmp]# cat /proc/1580889/stack
[<0>] poll_schedule_timeout.constprop.14+0x42/0x70
[<0>] do_sys_poll+0x493/0x520
[<0>] __x64_sys_poll+0x37/0x130
[<0>] do_syscall_64+0x5b/0x1a0
[<0>] entry_SYSCALL_64_after_hwframe+0x65/0xca
```

<!--more-->

实际上，在`podman exec`执行过程中，命令进程是由`runc`进程创建的，但因为`conmon`实现中设置了`PR_SET_CHILD_SUBREAPTER`, 因而，`runc`进程退出后，命令进程的父进程就变为了`conmon`进程，由`conmon`进程负责回收退出的进程。
```c
	/*
	 * Set self as subreaper so we can wait for container process
	 * and return its exit code.
	 */
	int ret = prctl(PR_SET_CHILD_SUBREAPER, 1, 0, 0, 0);
	if (ret != 0) {
		pexit("Failed to set as subreaper");
	}
```

## 环境信息

| 项目 | 详情 |
|------|------|
| OS | CentOS 8 (kernel 4.18.0-348.el8.x86_64) |
| Podman | 3.3.1 |
| conmon | 2.0.29 |
| GLib | 2.56.4 |

## 根因

每个进程都有一个信号掩码（signal mask），它是一个位集合，决定了哪些信号当前被阻塞（blocked）。被阻塞的信号不会被递送给进程，而是挂起（pending），直到掩码中取消阻塞后才会处理。进程的sigmask是从父进程继承, 而且`exec`时并不会重置。而我们的程序中阻塞了`SIGUSR1`，`conmon 2.0.29`则依赖`SIGUSR1`信号唤醒机制, 这样导致`GLib`主循环永久阻塞在 `poll()`，无法回收已退出的子进程。那么为什么不是每次都产生僵尸进程呢? 这是因为`conmon`在实际开始主循环前，会运行一次回收子进程的逻辑。因而只要子进程退出在主循环开始阻塞之前运行，就可以正常退出。


## 调用链

```
我们的进程(SigBlk=0x5200, 含 SIGUSR1)
  → popen → /bin/sh (继承 SIGUSR1 阻塞)
    → podman exec <cid> <cmd>
      → fork/exec conmon (继承 SIGUSR1 阻塞)
        → conmon.c:202-205: sigprocmask(SIG_BLOCK, {SIGTERM,SIGQUIT,SIGINT}, &oldmask)
           ↳ oldmask = 继承的掩码, SIGUSR1 继续阻塞
        → conmon.c:289: sigprocmask(SIG_SETMASK, &oldmask, NULL)
           ↳ 恢复 oldmask，SIGUSR1 仍然阻塞
        → conmon.c:304: g_unix_signal_add(SIGUSR1, on_sigusr1_cb, &data)
        → conmon.c:306: signal(SIGCHLD, on_sigchld)
        → fork/exec runc (创建 exec 会话)
        → conmon.c:436: g_idle_add(check_child_processes_cb, &data)
        → conmon.c:437: g_main_loop_run(main_loop) — 进入主循环
        → 子进程退出 → SIGCHLD → on_sigchld → raise(SIGUSR1)
           ↳ SIGUSR1 被阻塞 → Pending 永存
           ↳ poll() 无 fd 就绪 → 永久卡死
```

## GLib 主循环机制

理解`conmon`的卡死现象，需要先理解`GLib 2.56.4`的主循环运行机制。`GLib`主循环由 `GMainLoop` 和 `GMainContext` 两部分组成。`GMainContext` 维护一个事件源（`GSource`）链表，每种事件源实现 `prepare`、`query`、`check`、`dispatch` 四个虚函数。`g_main_loop_run()` 本质上是循环调用 `g_main_context_iterate(context, TRUE)`，每次迭代执行以下四个阶段：

```
g_main_loop_run(main_loop)
  → g_main_context_iterate(context, TRUE)        // blocking=TRUE
      ├─ g_main_context_prepare(context, &max_priority)
      │   遍历所有 source，调用 source->prepare()
      │   若 prepare() 返回 TRUE → max_priority = max(max_priority, source->priority)
      │
      ├─ g_main_context_query(context, max_priority, &timeout, fds, n_fds)
      │   遍历所有 source，调用 source->query() 收集 fd 到 poll 数组
      │   计算 timeout：
      │     - 若有就绪的 source（max_priority >= G_PRIORITY_DEFAULT_IDLE）→ timeout=0
      │     - 若有定时器 → timeout = min(timeout, 最近定时器剩余时间)
      │     - 否则 → timeout = -1（无限等待）
      │
      ├─ g_main_context_poll(context, timeout, priority, fds, n_fds)
      │   底层调用 poll(fds, n_fds, timeout)
      │   ★ 注意：是 poll()，不是 ppoll()，不接收信号掩码参数
      │
      ├─ g_main_context_check(context, max_priority, fds, n_fds)
      │   遍历所有 source，调用 source->check() 判断哪些 fd 已就绪
      │
      └─ g_main_context_dispatch(context)
          遍历所有 source，对已就绪的 source 调用 source->dispatch()
```

### timeout 的计算逻辑

`g_main_context_query()` 中 timeout 的计算是理解问题的关键：

```c
static gint
g_main_context_query(GMainContext *context,
                     gint max_priority,
                     gint *timeout,
                     GPollFD *fds,
                     gint n_fds)
{
    gint n_poll = 0;
    *timeout = -1;  // 默认：无限等待

    for (source = context->source_list; source; source = source->next) {
        if (source->priority <= max_priority) {
            // 收集 fd
            n_poll += source->source_funcs->query(source, ...);

            // 若有就绪 source（prepare 返回 TRUE 且优先级不高于 max_priority）
            if (SOURCE_READY(source, max_priority))
                *timeout = 0;  // 不阻塞，立即返回 dispatch
        }
    }

    // 检查最近定时器
    if (timer_timeout != -1)
        *timeout = MIN(*timeout, timer_timeout);  // -1 和正数取 min = 正数

    return n_poll;
}
```

**timeout 的三种取值**：
| timeout 值 | 含义 | poll 行为 |
|-----------|------|----------|
| `0` | 有待处理的 source，立即返回 | `poll(fds, nfds, 0)` — 不阻塞 |
| `>0` | 最近的定时器到期时间（毫秒） | `poll(fds, nfds, timeout)` — 定时阻塞 |
| `-1` | 无定时器、无就绪 source | `poll(fds, nfds, -1)` — **无限阻塞** |

### Signal Source 的工作原理

`g_unix_signal_add(SIGUSR1, on_sigusr1_cb, &data)` 内部创建一个 GLib signal source：

1. GLib 创建一对 `eventfd`（或 pipe），启动一个内部 worker 线程
2. Worker 线程阻塞在 `sigwaitinfo()` 上，等待目标信号
3. 当目标信号被投递，worker 线程收到后向 eventfd 写入数据
4. eventfd 的读端被注册到主循环的 poll fd 集合中
5. 主循环 `poll()` 看到 eventfd 可读 → check 返回 TRUE → dispatch 调用 `on_sigusr1_cb`

**关键约束**：`sigwaitinfo()` 只能捕获未被阻塞的信号。如果 SIGUSR1 在线程的信号掩码中被阻塞，worker 线程永远收不到它，eventfd 永远不会变为可读。

### Idle Callback 的工作原理

`g_idle_add(check_child_processes_cb, &data)` 创建一个优先级为 `G_PRIORITY_DEFAULT_IDLE`（200）的 idle source：

- `prepare()` 返回 TRUE（表示 source 总是就绪）
- `query()` 不注册任何 fd
- `check()` 返回 TRUE
- `dispatch()` 调用用户回调
- 回调返回 `G_SOURCE_REMOVE` 后，source 被自动销毁

`conmon`进入主循环前会注册一个 idle callback（`conmon.c:436-437`）：

```c
/* conmon.c:435-438 */
if (opt_api_version < 1 || !opt_exec || !opt_terminal || container_status < 0) {
    g_idle_add(check_child_processes_cb, &data);   // line 436
    g_main_loop_run(main_loop);                     // line 437
}
```

对于 `podman exec`（`opt_exec=1`, `opt_api_version>=1`）：
- 无终端模式（`opt_terminal=0`）：始终进入主循环
- 有终端模式（`opt_terminal=1`）：仅当 `container_status < 0`（子进程尚未被回收）时进入

Idle callback 在主循环的 **dispatch 阶段**执行，发生在 `poll()` 返回之后。它的逻辑就是回收退出子进程，因为`check_child_processes_cb`中返回`G_SOURCE_REMOVE`, 因而这个idle callback是**一次性**的 —— 执行后立即移除。

根据源码，`conmon`回收子进程有两个路径:

### 路径一：idle callback 捕获子进程

这是绝大多数正常场景下的退出路径，利用 idle callback 在主循环第一次迭代即回收子进程：

```
g_main_loop_run(main_loop)
  │
  ├─ 第 1 次迭代:
  │   ├─ g_main_context_prepare:
  │   │    idle source->prepare() → TRUE
  │   │    max_priority = max(-1, G_PRIORITY_DEFAULT_IDLE) = 200
  │   │
  │   ├─ g_main_context_query:
  │   │    遍历所有 source:
  │   │      - idle source: SOURCE_READY = TRUE（priority 200 <= max_priority 200）
  │   │        → *timeout = 0
  │   │      - signal source (SIGUSR1): 注册 eventfd 到 fds
  │   │      - stdio sources: 注册 mainfd_stdout/stderr 到 fds
  │   │    无定时器 → timeout 保持 0
  │   │    n_poll = 3（eventfd + stdout + stderr 共 3 个 fd）
  │   │
  │   ├─ g_main_context_poll:
  │   │    poll([eventfd, stdout, stderr], nfds=3, timeout=0)  ← 立即返回
  │   │
  │   ├─ g_main_context_check: 检查哪些 fd 就绪
  │   │
  │   └─ g_main_context_dispatch:
  │        idle source 就绪 → 调用 check_child_processes_cb(&data)
  │          └─ check_child_processes(pid_to_handler, exit_status_cache)
  │               └─ waitpid(-1, &status, WNOHANG)
  │                    ├─ 返回 pid>0 → 调用 container_exit_cb:
  │                    │    container_status = status
  │                    │    container_pid = -1
  │                    │    g_main_loop_quit(main_loop)  ← 退出主循环 ✅
  │                    │
  │                    └─ 返回 0 → 子进程还在运行，继续（见路径二）
  │        check_child_processes_cb 返回 G_SOURCE_REMOVE → idle source 被销毁
  │
  └─ （如果 container_exit_cb 已调用 g_main_loop_quit）
     主循环退出，conmon 正常结束
```

**这条路径的关键前提**：idle callback 执行时，子进程已经退出。idle callback 的执行时刻是第一次 poll 返回之后的 dispatch 阶段，与子进程创建之间仅有微秒级的间隔。对于执行速度极快的命令（如 `md5sum /bin/cat`子进程大概率已经退出，`waitpid` 可以成功回收。

### 路径二：idle callback 未找到子进程 + SIGUSR1 被阻塞（卡死路径）

当 idle callback 执行时子进程尚未退出，主循环将进入第二次迭代，走向永久阻塞：

```
g_main_loop_run(main_loop)
  │
  ├─ 第 1 次迭代:
  │   ├─ prepare: idle source 就绪 → max_priority = 200
  │   ├─ query: timeout = 0（idle source 存在）
  │   ├─ poll([eventfd, stdout, stderr], nfds=3, timeout=0)  ← 立即返回
  │   └─ dispatch: check_child_processes_cb()
  │        └─ waitpid(-1, &status, WNOHANG)
  │             └─ 返回 0 ← 子进程还在运行！
  │        check_child_processes_cb 返回 G_SOURCE_REMOVE → idle source 销毁
  │
  └─ 第 2 次迭代:
      ├─ prepare: 无 idle source → max_priority = -1
      │
      ├─ query:
      │   遍历 source 列表：
      │     - signal source (SIGUSR1): eventfd 未就绪 → 不设置 timeout=0
      │     - stdio source (stdout): 可能还有数据 → 注册 fd
      │     - stdio source (stderr): 可能还有数据 → 注册 fd
      │   无定时器、无 idle source → timeout = -1  ← 无限等待！
      │   n_poll = 3
      │
      ├─ poll([eventfd, stdout, stderr], nfds=3, timeout=-1)  ← 无限阻塞
      │   │
      │   │  ... 子进程退出 ...
      │   │  内核向 conmon 发送 SIGCHLD
      │   │  → on_sigchld() 被调用
      │   │    → raise(SIGUSR1)     ← 期望唤醒主循环
      │   │      │
      │   │      ├─ SIGUSR1 未阻塞 → 信号投递 → worker 线程 sigwaitinfo 收到
      │   │      │   → eventfd 可写 → poll 返回 → dispatch on_sigusr1_cb ✅
      │   │      │
      │   │      └─ SIGUSR1 被阻塞 → 信号置为 Pending
      │   │          → worker 线程 sigwaitinfo 永远收不到
      │   │          → eventfd 永远不会可读
      │   │          → poll() 没有 fd 就绪
      │   │          → 永久卡死 💀
      │   │
      │   └─ （永不返回）
      │
      子进程状态：[defunct]（已退出但无人 waitpid）
      conmon 状态：wchan=__x64_sys_poll, SigBlk=0x200, SigPnd=0x200
```

**卡死场景的完整时序**：

```
时间线:
 T0      conmon fork/exec 子进程（如 sleep 5）
 T0+μs   conmon 进入主循环
 T0+μs   第 1 次迭代：idle callback 执行，waitpid 返回 0（子进程还在跑）
 T0+μs   第 2 次迭代：poll(fds, nfds=3, timeout=-1) — 无限等待
 ...
 T0+5s   子进程 (sleep 5) 退出 → SIGCHLD 发送给 conmon
         on_sigchld() → raise(SIGUSR1)
         SIGUSR1 被阻塞 → SigPnd 置位，永不投递
         poll() 无 fd 就绪 → 永远不会返回
         子进程变为 zombie (defunct)
```

Linux 的 `fork()` 和 `execve()` 都不会重置进程的信号掩码（`execve` 会将信号处理器重置为 `SIG_DFL`，但掩码保留）。因此我们进程阻塞的 SIGUSR1 会一直传递到 conmon 进程。

### conmon 的信号掩码管理缺陷

conmon 在 `conmon.c:202-205` 管理信号掩码，但从不处理 SIGUSR1：

```c
/* conmon.c:202-205 */
sigset_t mask, oldmask;
sigemptyset(&mask);
sigaddset(&mask, SIGTERM);
sigaddset(&mask, SIGQUIT);
sigaddset(&mask, SIGINT);
sigprocmask(SIG_BLOCK, &mask, &oldmask);
// oldmask = 继承自父进程的掩码（含 SIGUSR1）  ← SIGUSR1 仍然被阻塞！

/* fork runc 子进程 ... */

/* conmon.c:289 */
sigprocmask(SIG_SETMASK, &oldmask, NULL);
// 恢复 oldmask → SIGUSR1 仍然被阻塞！  ← 这里并未解除 SIGUSR1
```

conmon 的 `SIG_BLOCK` + `SIG_SETMASK` 操作只涉及 `{SIGTERM, SIGQUIT, SIGINT}`，对 SIGUSR1 的阻塞状态既不感知也不修改。从父进程继承的 SIGUSR1 阻塞状态被原封不动地保留。

## POC 最小复现

根据实现原理，就可以实现一个最小的触发程序:

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    // Step 1: 阻塞 SIGUSR1
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGUSR1);
    sigprocmask(SIG_BLOCK, &set, NULL);

    // Step 2: 在容器中执行慢命令
    //         子进程必然在 idle callback 之后退出 → 触发 SIGUSR1 路径
    FILE *fp = popen("podman exec <container_id> sleep 5", "r");
    if (fp) {
        char buf[256];
        while (fgets(buf, sizeof(buf), fp)) {}
        pclose(fp);
    }

    // popen 将永久阻塞 — conmon 卡死在 poll()
    printf("This line will never be reached\n");
    return 0;
}
```

**触发条件仅需两个**：
1. 父进程阻塞 SIGUSR1（`sigprocmask(SIG_BLOCK, {SIGUSR1})`）
2. 容器内命令执行时间 > idle callback 窗口

## 实际修复 commit：d91cc4321（巧合修复）

官方commit[`d91cc4321797eaa84dcfb7863e91632d2fe26861`](https://github.com/containers/conmon/commit/d91cc4321797eaa84dcfb7863e91632d2fe26861)，commit message：

```plain
Stop using g_unix_signal_add() to avoid threads
g_unix_signal_add() is implemented via an implicitly-created worker
thread in GLib.  This thread creates an eventfd, and conflicts with
conmon's general approach to taking full ownership of all open fds in
the process.

Stop using g_unix_signal_add() and switch to using g_unix_fd_add() on a
signalfd() as an alternative approach.

Fixes #330

Signed-off-by: Allison Karlitskaya <allison.karlitskaya@redhat.com>
```

根据`commit`消息，官方这个修复的目的其实与我们的问题无关, 是因为GLib 的 `g_unix_signal_add()` 内部创建隐式 worker 线程（创建 eventfd），与 conmon "接管所有 fd 避免泄漏" 的策略冲突, 换用 signalfd 消除多余线程。但修改之后，`conmon`不再依赖`SIGUSR1`信号，因而巧合的修复了这个卡死的问题。
