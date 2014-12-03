---
layout: post
title: "Passenger导致NGINX master进程阻塞定位及解决"
date: 2014-12-03 11:16:33 +0800
comments: true
categories: PASSENGER
---
我们线上服务使用nginx+passenger-3.0.11来运行Rails程序。我们发现当执行若干次`nginx -s reload`或者kill -HUP `cat /usr/local/nginx/logs/nginx.pid`之后，再执行以上命令不再生效。

用pstack观察master进程，master进程阻塞在read()操作上。
{% img /images/2014-12-03/1.png %}
使用gcore生成core文件，然后用gdb查看，查看栈帧所在文件及行号。
{% img /images/2014-12-03/2.png %}
查看passenger源码。readExact()逻辑为一直读到size大小返回。readArrayMessage()代码逻辑为先从传入fd中读取两个字节做为长度，再从fd中读取相应长度的内容。Passenger::AgentsStarter::start函数的过程为创建一个socketpair，然后fork()子进程，子进程执行PassengerWatchdog。父子进程通过该socketpair进行通信。master就是卡在读取socketpair上。

从gdb中看到readExact()传入的size为12848，即从socketpair中读到的长度为12848，而实际读取的内容长度为621，并且读取的内容也很诡异。应该是某个地方向socketpair中写入了错误内容。
{% img /images/2014-12-03/3.png %}
继续查看passenger代码中子进程的逻辑。关闭socketpair一端。然后将socketpair另一端通过dup2()调用复制到3上。然后关闭除了0-3以外的所有文件描述符。接着解除信号阻塞，重置信号处理函数。

通过strace追踪master进程及其子进程行为，发现子进程确实发送了错误内容。"20"的十六进制表示为0x3230，按网络字节序读出正好是12848。从调用位置看是在SIGCHLD的信号处理函数中。而从写入内容看像是nginx在写日志。
{% img /images/2014-12-03/4.png %}
继续追查为什么会有SIGCHLD产生。最后发现getHighestFileDescriptor()函数中子进程会fork()出孙进程，获取当前打开的最大fd，子进程等待孙进程退出后返回。由于孙进程退出，子进程收到了SIGCHLD信号。但是由于子进程是由master进程fork()出来的，SIGCHLD信号是被阻塞的。当执行sigprocmask()时，被阻塞的SIGCHLD被处理了，而这时的信号处理函数是ngx_signal_handler(), 该函数会调用ngx_log_error()向error log写入日志。若error log的fd为3，就发生我们上述的情况。

为了避免这种情况，应该先重置信号处理函数，再解除信号阻塞。

patch:

https://github.com/flygoast/passenger/commit/0b62808943aba65432f0b492f4ef941499fad02c
