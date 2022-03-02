---
title: SOCKS5和Dante介绍
date: 2022-03-02 23:23:58
tags:
- SOCKS5
categories: Network
description:
---
`SOCKS`是一个比较简单的通用代理协议，用于在客户端与服务器之间代理网络数据包。最新的版本是`5`, 所以一般叫做`SOCKS5`，协议规范是[RFC1928](https://datatracker.ietf.org/doc/html/rfc1928)。但`SOCKS5`并不兼容之前的`SOCKS4`和`SOCKS4A`。`SOCKS5`在`SOCKS4`的基础上添加了`UDP`转发功能和`校验`机制。

`SOCKS5`的工作过程简单可以归纳为协商、请求、和转发三个阶段。以`TCP`代理场景来看, 一般流程是:

1. 客户端建立TCP连接
2. 客户端发送客户端侧支持的校验方法
3. 服务端回应选择的校验方法
4. 客户端和服务端之间按选择的校验方法完成校验
5. 客户端发送所需的请求给服务端。`SOCKS5`支持不同的请求类型，包括`CONNECT`, `BIND`, `UDPASSOCIATE`等。
6. 服务端接收到请求，从中解析出目的地址，建立到目的地址的连接。
7. 发送成功信息给客户端。
8. 客户端开始发送应用层信息。
9. 服务端在客户端和目的地址之间转发应用层信息。

其中，2-4完成协商阶段，5-7完成请求阶段，之后进入转发阶段。

当前常用的校验方法是`USERNAME/PASSWORD`([RFC1929](https://datatracker.ietf.org/doc/html/rfc1929))和`GSS-API`([RFC1961](https://datatracker.ietf.org/doc/html/rfc1961))，具体的校验过程是由不同校验方法来自定义的。

使用`USERNAME/PASSWORD`校验方法的`TCP`代理时序图如下:
{% img /images/2022-03-02/1.png %}

<!--more-->

[Dante](https://www.inet.no/dante/index.html)是个较为成熟的`SOCKS`服务器开源实现，一些高级模块的代码没有开源，需要购买商业支持。但`Dante`的整体代码逻辑比较清晰，还是比较容易扩展的。

`Dante`的进程结构比较有意思，以一种类似流水线的方式在多个进程间传递并处理请求, 而且会根据请求量对相应进程数量进行调节，相应进程的处理量会通过修改进程名称来体现。

请求传递具体实现上依赖通过`UNIX Socket`在进程间传递`文件描述符`，机制可以参考这篇文章:
* https://copyconstruct.medium.com/file-descriptor-transfer-over-unix-domain-sockets-dcbbf5b3b6ec


`Dante`包括`5`种不同角色的进程:
`mother`: 初始启动的主进程，其他的进程都由`mother`进程创建。但在`fork`其他进程前，`mother`会建立两对`UNIX Socket`通道，一个用于在父子进程之间发送数据和文件描述符，另一个用于监控进程的存活状态。创建完其他进程后，`mother`开始循环处理客户端的TCP连接的`accept`。
`monitor`: 基于超时时间处理其他进程的共享内存数据。
`negotiate`: 处理`SOCKS`协议的协商。
`request`: 建立到远端服务器的请求。
`io`: 在客户端和远端服器之间转发数据。

整体的进程架构图如下:
{% img /images/2022-03-02/2.png %}

上述的基于`TCP`的代理流程在`sockd`中的处理流程是:
* 客户端向`sockd`发起`TCP`连接。
* `sockd`的`mother`进程`accept`连接，然后选择一个`negotiate`进程，将相关数据结构及客户端连接`fd`发送给它。
* `negotiate`进程接收到`fd`和数据结构，从客户端`fd`读取协商请求和方法。从中选定一种方法，发送给客户端。
* 客户端根据`sockd`返回的校验方法发送校验请求。
* `negotiate`进程读取校验请求并校验通过后，发送成功状态给客户端。
* 客户端发送代理请求。
* `negotiate`进程读取代理请求并构建相应的数据结构，将这个数据结构和客户端`fd`再发送回`mother`进程。
* `mother`进程收到这些信息后再选择一个`request`进程，并将这些信息转发给该进程。
* `request`进程根据收到的请求信息建立到远端服务器的TCP连接，并根据客户端和远端服务器两端信息构建相关数据结构，和两个`fd`一起发送回`mother`进程。
* `mother`进程将收到的数据和两个`fd`转发给选定的一个`io`进程。
* `io`进程接收到这些信息后，在两个`fd`之间进行数据转发。

整体流程如图:
{% img /images/2022-03-02/3.png %}

每个角色进程的逻辑主体都是基于`select`实现的事件驱动模型。众所周知，`select`有最大支持`1024`个文件描述符的限制。而在一些操作系统具体实现上，实际并不会真的去检查该限制。`Dante`的实现就利用这一点来突破`1024`的限制。这篇文章介绍了这种方法:
https://blog.csdn.net/dog250/article/details/105896693

但实际上这里还是存在一个BUG。`Dante`默认的编译情况下，会定义`_FORTIFY_SOURCE`, 这会导致`FD_SET`中会去检查`1024`的限制。当代理请求较多时，子进程越来越多，`mother`的`fd`数量就会超过`1024`而导致进程崩溃。

可以通过不再定义这个宏来修复问题，也可以通过另外的方法来绕过这个问题。

`Dante`支持创建多个`mother`进程，每个`mother`都会再创建自己的一系列`negotiate`,`request`, `io`等进程，但`monitor`进程只是最初的`mother`(叫做`main mother`)会创建。因为`main mother`是在开始`listen`之后再`fork`其他进程，因而多个`mother`进程都监听在同一端口上，它们会调用`accept`来抢夺客户端连接。抢夺到连接的`mother`会在自己的一系列子进程中进行处理。这样我们可以限制每个`mother`的文件描述符小于`1024`,而启动若干个`mother`。`mother`的数量可以通过命令行参数`-N`来指定。这种情况的进程架构如图:
{% img /images/2022-03-02/4.png %}

`Dante`里很多实现细节不太考虑性能, 大量使用数组和遍历。举例来说，`negotiate`进程每次循环会调用`neg_gettimedout`函数获取一个超时的协商请求:
```c
while (1 /* CONSTCOND */) {
    ...


#if HAVE_NEGOTIATE_PHASE
    gettimeofday_monotonic(&tnow);
    while ((neg = neg_gettimedout(&tnow)) != NULL) {

    }
}
```
而`neg_gettimedout`的实现里依然是一个遍历:
```c
static sockd_negotiate_t *
neg_gettimedout(const struct timeval *tnow)
{
   size_t i;

   for (i = 0; i < negc; ++i) {
      if (!negv[i].allocated
      || CRULE_OR_HRULE(&negv[i])->timeout.negotiate == 0)
         continue;

      if (socks_difftime(tnow->tv_sec, negv[i].state.time.negotiatestart.tv_sec)
      >= CRULE_OR_HRULE(&negv[i])->timeout.negotiate)
         return &negv[i];
   }

   return NULL;
}
```
当协商请求很多时，`negv`这个数组基本被分配完，如果有一个请求超时，`neg_gettimedout`返回超时请求，下一次依然从数组第一个元素开始遍历。

如果对于`SOCKS`服务器性能有比较强的需求，`Dante`还是有很多优化空间的。比如使用`epoll`替换`select`, 使用其他结构替换数组等等。

参考:
* https://p0lycarp.github.io/2019/fortify/
* https://en.wikipedia.org/wiki/SOCKS
* https://datatracker.ietf.org/doc/html/rfc1928
* https://datatracker.ietf.org/doc/html/rfc1929
* https://datatracker.ietf.org/doc/html/rfc1961
* https://blog.csdn.net/dog250/article/details/105896693
* https://www.inet.no/dante/index.html
* https://securityintelligence.com/posts/socks-proxy-primer-what-is-socks5-and-why-should-you-use-it/
* https://zhuanlan.zhihu.com/p/65094630
* https://blog.csdn.net/JassFuchang/article/details/7170321
* https://wiyi.org/socks5-protocol-in-deep.html
* https://www.rapidseedbox.com/blog/guide-to-socks5-proxy
* https://copyconstruct.medium.com/file-descriptor-transfer-over-unix-domain-sockets-dcbbf5b3b6ec
* https://cybarrior.com/blog/2019/03/07/dante-socks5-proxy-server-setup/
