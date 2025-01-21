---
title: 多线程程序设置CURLOPT_NOSIGNAL选项
date: 2025-01-21 09:35:12
categories: MISC
---
近期遇到一个C++程序退出的问题，经过调查发现，程序是由于接收到`SIGPIPE`信号而退出。该程序是多线程程序，使用`libcurl`进行`HTTPS`访问，同时设置了`CURLOPT_NOSIGNAL`选项，但没有自己处理`SIGPIPE`信号。在一些情况下，连接已经处理关闭状态，但应用程序不知道，依然向连接发送数据，就会导致`SIGPIPE`信号产生，进而导致程序退出。修复方法比较简单，只要应用程序设置`SIGPIPE`信号的处理程序即可。

上述问题要发生的一个前提是`CURLOPT_NOSIGNAL`选项需要被设置，什么情况下需要设置它呢？`libcurl`的作者的博文上写到过这个问题:

*   [https://daniel.haxx.se/blog/2021/09/27/common-mistakes-when-using-libcurl/](https://daniel.haxx.se/blog/2021/09/27/common-mistakes-when-using-libcurl/)

```text-plain
12. Understanding CURLOPT_NOSIGNAL
Signals is a Unix concept where an asynchronous notification is sent to a process or to a specific thread within the same process in order to notify it of an event that occurred.

What does libcurl use signals for?
When using the synchronous name resolver, libcurl uses alarm() to abort slow name resolves (if a timeout is set), which ultimately sends a SIGALARM to the process and is caught by libcurl

By default, libcurl installs its own sighandler while running, and restores the original one again on return – for SIGALARM and SIGPIPE.

Closing TLS (with OpenSSL etc) can trigger a SIGPIPE if the connection is dead.

Unless CURLOPT_NOSIGNAL is set! (default)

What does CURLOPT_NOSIGNAL do?
It prevents libcurl from triggering signals

When disabled, it prevents libcurl from installing its own sighandler and…

Generated signals must then be handled by the libcurl-using application itself
```
<!--more-->

解释的比较清楚。`libcurl`在使用同步DNS解析时，使用的是`alarm()`函数来实现超时。我们从源码可以看一下, 这里参考的是`curl-7.52.0`, 位于源码文件`lib/hostip.c`中函数`Curl_resolv_timeout`：

```text-plain
627     /*************************************************************
628      * Set signal handler to catch SIGALRM
629      * Store the old value to be able to set it back later!
630      *************************************************************/
631 #ifdef HAVE_SIGACTION
632     sigaction(SIGALRM, NULL, &sigact);
633     keep_sigact = sigact;
634     keep_copysig = TRUE; /* yes, we have a copy */
635     sigact.sa_handler = alarmfunc;
636 #ifdef SA_RESTART
637     /* HPUX doesn't have SA_RESTART but defaults to that behaviour! */
638     sigact.sa_flags &= ~SA_RESTART;
639 #endif
640     /* now set the new struct */
641     sigaction(SIGALRM, &sigact, NULL);
642 #else /* HAVE_SIGACTION */
643     /* no sigaction(), revert to the much lamer signal() */
644 #ifdef HAVE_SIGNAL
645     keep_sigact = signal(SIGALRM, alarmfunc);
646 #endif
647 #endif /* HAVE_SIGACTION */
648
649     /* alarm() makes a signal get sent when the timeout fires off, and that
650        will abort system calls */
651     prev_alarm = alarm(curlx_sltoui(timeout/1000L));
```

`libcurl`使用`alarm()`函数设置超时，并且使用配置了自身的信号处理函数。但如果程序是多线程环境，并不能保证是由调用`alarm()`函数的线程来处理信号。这会导致程序异常行为。

内核中选择处理信号的线程是由函数`complete_signal`实现，这里参考`CentOS`内核版本: `3.10.0-957.el7.x86_64`的代码:

```text-plain
static void complete_signal(int sig, struct task_struct *p, int group)
{
    struct signal_struct *signal = p->signal;
    struct task_struct *t;

    /*
     * Now find a thread we can wake up to take the signal off the queue.
     *
     * If the main thread wants the signal, it gets first crack.
     * Probably the least surprising to the average bear.
     */
    if (wants_signal(sig, p))
        t = p;
    else if (!group || thread_group_empty(p))
        /*
         * There is just one thread and it does not need to be woken.
         * It will dequeue unblocked signals before it runs again.
         */
        return;
    else {
        /*
         * Otherwise try to find a suitable thread.
         */
        t = signal->curr_target;
        while (!wants_signal(sig, t)) {
            t = next_thread(t);
            if (t == signal->curr_target)
                /*
                 * No thread needs to be woken.
                 * Any eligible threads will see
                 * the signal in the queue soon.
                 */
                return;
        }
        signal->curr_target = t;
    }
    

    /*
     * Found a killable thread.  If the signal will be fatal,
     * then start taking the whole group down immediately.
     */
    if (sig_fatal(p, sig) &&
        !(signal->flags & (SIGNAL_UNKILLABLE | SIGNAL_GROUP_EXIT)) &&
        !sigismember(&t->real_blocked, sig) &&
        (sig == SIGKILL || !t->ptrace)) {
        /*
         * This signal will be fatal to the whole group.
         */
        if (!sig_kernel_coredump(sig)) {
            /*
             * Start a group exit and wake everybody up.
             * This way we don't have other threads
             * running and doing things after a slower
             * thread has the fatal signal pending.
             */
            signal->flags = SIGNAL_GROUP_EXIT;
            signal->group_exit_code = sig;
            signal->group_stop_count = 0;
            t = p;
            do {
                task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK);
                sigaddset(&t->pending.signal, SIGKILL);
                signal_wake_up(t, 1);
            } while_each_thread(p, t);
            return;
        }
    }

    /*
     * The signal is already in the shared-pending queue.
     * Tell the chosen thread to wake up and dequeue it.
     */
    signal_wake_up(t, sig == SIGKILL);
    return;
}
```

根据内核代码，可以确定对于多线程程序，会优先判断是否可以由主线程来处理信号。

并且，`libcurl`中使用`sigsetjmp/siglongjmp`实现`SIGALRM`的处理。当由其他线程来处理`SIGALRM`信号，并执行`siglongjmp`之后，整个线程的执行逻辑就被破坏了，程序会崩溃。

```text-plain
#ifdef USE_ALARM_TIMEOUT
/*
* This signal handler jumps back into the main libcurl code and continues
* execution.  This effectively causes the remainder of the application to run
* within a signal handler which is nonportable and could lead to problems.
*/
static
RETSIGTYPE alarmfunc(int sig)
{
 /* this is for "-ansi -Wall -pedantic" to stop complaining!   (rabe) */
 (void)sig;
 siglongjmp(curl_jmpenv, 1);
 return;
}
#endif /* USE_ALARM_TIMEOUT */
```

为了验证上面内容，修改一下`libcurl`代码，在日志输出中添加上`task id`, `patch`文件如下:

```text-plain
[root@localhost curl-curl-7_52_0]# diff -ruNp lib/hostip.c.orig lib/hostip.c
--- lib/hostip.c.orig	2025-01-20 21:31:41.370112869 +0800
+++ lib/hostip.c	2025-01-20 21:30:19.317253722 +0800
@@ -22,6 +22,7 @@

 #include "curl_setup.h"

+#include <syscall.h>
 #ifdef HAVE_NETINET_IN_H
 #include <netinet/in.h>
 #endif
@@ -619,7 +620,7 @@ int Curl_resolv_timeout(struct connectda
      before we invoke Curl_resolv() (and thus use "volatile"). */
   if(sigsetjmp(curl_jmpenv, 1)) {
     /* this is coming from a siglongjmp() after an alarm signal */
-    failf(data, "name lookup timed out");
+    failf(data, "%d: name lookup timed out", syscall(SYS_gettid));
     rc = CURLRESOLV_ERROR;
     goto clean_up;
   }
@@ -646,6 +647,8 @@ int Curl_resolv_timeout(struct connectda
 #endif
 #endif /* HAVE_SIGACTION */

+    infof(conn->data, "%d: set alarm\n", syscall(SYS_gettid));
+
     /* alarm() makes a signal get sent when the timeout fires off, and that
        will abort system calls */
     prev_alarm = alarm(curlx_sltoui(timeout/1000L));
```

修改代码后，编译使用同步DNS解析的静态库:

```text-plain
./buildconf
./configure --with-ssl --disable-threaded-resolver --enable-static --without-zlib
make
```

下面是验证程序源码，主线程调用`sleep()`, 再创建一个线程调用`libcurl`执行DNS解析:

```text-plain
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <syscall.h>
#include <signal.h>
#include <pthread.h>
#include <curl/curl.h>

pid_t gettid(void) {
    return syscall(SYS_gettid);
}

void *t1_main(void *arg) {
    printf("pthread1 started: %d\n", gettid());

    while (1) {
        CURLcode res;
        CURL *curl = curl_easy_init();
        if (!curl) {
            printf("curl_easy_init failed\n");
            exit(-1);
        }

        curl_easy_setopt(curl, CURLOPT_URL, "http://nosuchdomain.com/");
        curl_easy_setopt(curl, CURLOPT_VERBOSE, 1L);
        curl_easy_setopt(curl, CURLOPT_CONNECTTIMEOUT, 5L);

        res = curl_easy_perform(curl);

        printf("curl_easy_perform: %d, %s\n", res, curl_easy_strerror(res));

        curl_easy_cleanup(curl);

        sleep(1);
    }
}

int main(int argc, char **argv) {
    int ret;
    pthread_t p1;

    curl_global_init(CURL_GLOBAL_DEFAULT);

    curl_version_info_data *version = curl_version_info(CURLVERSION_NOW);
    if (version->features & CURL_VERSION_ASYNCHDNS) {
        printf("Asynchoronous DNS\n");
        if (!version->age || version->ares_num) {
            printf("  C-ares\n");
        } else {
            printf("  threaded\n");
        }
    } else {
        printf("Synchoronous DNS\n");
    }

    printf("main started: %d\n", gettid());

    ret = pthread_create(&p1, NULL, t1_main, NULL);
    printf("create pthread1, ret: %d\n", ret);

    while (1) {
        sleep(3600);
        printf("main waked up\n");
    }

    curl_global_cleanup();
    return 0;
}
```

使用编译的`libcurl`编译验证程序:

```text-plain
gcc -g -O0 t.cpp -lpthread lib/.libs/libcurl.a
```

为了构造DNS解析超时，使用`iptables`将DNS响应丢失:

```text-plain
iptables -I INPUT -p udp --sport 53 -j DROP
```

执行验证程序:

```text-plain
[root@localhost curl-curl-7_52_0]# ./a.out
Synchoronous DNS
main started: 23755
create pthread1, ret: 0
pthread1 started: 23756
* 23756: set alarm
* 23755: name lookup timed out
* Couldn't resolve host 'nosuchdomain.com'
* Closing connection 0
curl_easy_perform: 6, Couldn't resolve host name
* 23755: set alarm
* 23755: name lookup timed out
* Couldn't resolve host 'nosuchdomain.com'
* Closing connection 0
curl_easy_perform: 6, Couldn't resolve host name
* 23755: set alarm
* 23755: name lookup timed out
* Couldn't resolve host 'nosuchdomain.com'
* Closing connection 0
curl_easy_perform: 6, Couldn't resolve host name
* 23755: set alarm
* 23755: name lookup timed out
* Couldn't resolve host 'nosuchdomain.com'
* Closing connection 0
curl_easy_perform: 6, Couldn't resolve host name
* 23755: set alarm
* 23755: name lookup timed out
* Couldn't resolve host 'nosuchdomain.com'
* Closing connection 0
curl_easy_perform: 6, Couldn't resolve host name
* 23755: set alarm
* 23755: name lookup timed out
* Couldn't resolve host 'nosuchdomain.com'
* Closing connection 0
curl_easy_perform: 6, Couldn't resolve host name
Segmentation fault (core dumped)
```

可以看到调用`alarm()`的线程为`23756`, 处理信号的线程却是主线程`23755`。收到`SIGALRM`信号的线程`23755`执行信号处理函数，通过`siglongjmp`开始执行原本应该是`23756`线程执行的代码。执行一段时间后程序崩溃。

由于这种原因，在多线程程序中使用同步DNS解析特性的`libcurl`库时，需要设置`CURLOPT_NOSIGNAL`选项。但这有一点副作用，就是DNS解析的超时机制就失效了:

```text-plain
  if(data->set.no_signal)
    /* Ignore the timeout when signals are disabled */
    timeout = 0;
  else
    timeout = timeoutms;

  if(!timeout)
    /* USE_ALARM_TIMEOUT defined, but no timeout actually requested */
    return Curl_resolv(conn, hostname, port, entry);
```

但该选项不只影响`SIGALRM`信号, 设置该选项后，`libcurl`对`SIGPIPE`信号也不再处理, 源码位`lib/transfer.c`的`Curl_pretransfer`函数:

```text-plain
1333 #if defined(HAVE_SIGNAL) && defined(SIGPIPE) && !defined(HAVE_MSG_NOSIGNAL)
1334     /*************************************************************
1335      * Tell signal handler to ignore SIGPIPE
1336      *************************************************************/
1337     if(!data->set.no_signal)
1338       data->state.prev_signal = signal(SIGPIPE, SIG_IGN);
1339 #endif
```

因而，这种情形下，使用`libcurl`的程序自身必须忽略`SIGPIPE`信号。这也就是开头所说的我们程序退出的原因。

另外，如果`libcurl`编译时指定了异步DNS解析，则`CURLOPT_NOSIGNAL`选项则没有设置的必要。`CentOS7`的`YUM`源中的`libcurl`实际都是异步DNS解析特性:

```text-plain
[root@localhost curl-curl-7_52_0]# curl-config  --feature
SSL
IPv6
unix-sockets
libz
AsynchDNS
IDN
NTLM
NTLM_WB
```

参考:

*   [https://suntus.github.io/2024/01/27/%E4%BF%A1%E5%8F%B7%E7%9B%B8%E5%85%B3/](https://suntus.github.io/2024/01/27/%E4%BF%A1%E5%8F%B7%E7%9B%B8%E5%85%B3/)
*   [https://www.antidebug.cn/get\_process\_signal\_info/](https://www.antidebug.cn/get_process_signal_info/)
*   [https://github.com/curl/curl/discussions/11324](https://github.com/curl/curl/discussions/11324)
*   [https://tianshuang.me/2022/12/RST/](https://tianshuang.me/2022/12/RST/)
*   [https://juejin.cn/post/7440852119014866953](https://juejin.cn/post/7440852119014866953)
*   [https://blog.csdn.net/weixin\_34248705/article/details/92216954](https://blog.csdn.net/weixin_34248705/article/details/92216954)
*   [https://ty-chen.github.io/linux-kernel-signal/](https://ty-chen.github.io/linux-kernel-signal/)
*   [https://daniel.haxx.se/blog/2021/09/27/common-mistakes-when-using-libcurl/](https://daniel.haxx.se/blog/2021/09/27/common-mistakes-when-using-libcurl/)
