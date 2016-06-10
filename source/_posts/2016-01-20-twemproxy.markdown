---
layout: post
title: "twemproxy实现分析"
date: 2016-01-20 00:36:43 +0800
comments: true
categories: twemproxy
---
twemproxy(https://github.com/twitter/twemproxy)是Twitter开源的Redis和Memcached代理程序，它可以将多个后端server组织成一个ServerPool, 基于请求的Key从Pool中选取一个server实例进行操作，从而实现分片存储。

twemproxy采用事件驱动处理网络数据收发。程序启动后会单独创建一个线程来处理stats请求，而主线程进入事件循环处理访问所有ServerPool的Redis或Memcached请求。

<!--more-->
下面分析一下请求的处理过程:

客户端建立到ServerPool的TCP连接后，该ServerPool的监听fd的读事件被触发，回调函数proxy_recv()调用proxy_accept()来处理请求。

首先，调用accept()接受连接:
```c
sd = accept(p->sd, NULL, NULL);
```

接着调用conn_get()为该连接分配一个conn结构。conn结构既用于表示客户端到twemproxy的连接(client connection)，也用于表示twemproxy到后端server的连接(server connection)，因而该结构针对两种不同类型的连接用函数指针来指向不同的回调函数来完成不同操作。

对于client connection, 用于接收和发送请求的函数指针为:
```c
conn->recv = msg_recv;
conn->recv_next = req_recv_next;
conn->recv_done = req_recv_done;

conn->send = msg_send;
conn->send_next = rsp_send_next;
conn->send_done = rsp_send_done;
```
而对于server connection, 这些函数指针则指向:
```c
conn->recv = msg_recv;
conn->recv_next = rsp_recv_next;
conn->recv_done = rsp_recv_done;

conn->send = msg_send;
conn->send_next = req_send_next;
conn->send_done = req_send_done;
```
之后，将接受的client connection的fd设置成非阻塞并注册到事件循环中。
```c
status = nc_set_nonblocking(c->sd);
...
status = event_add_conn(ctx->evb, c);
```
当客户端发送的Redis或Memcached请求到达后，client connection的读事件被触发，conn->recv指向的msg_recv()被调用，最终会调用msg_recv_chain()。该函数读取请求数据并调用msg_parse()解析请求。当解析出一个完整请求后，conn->recv_done指向的req_recv_done()函数被调用。

该函数的简化逻辑为:
```c
if (req_filter(ctx, conn, msg)) {
    return;
}

req_forward(ctx, conn, msg);
```
req_filter()用于判断是否需要将请求转发到后端。比如，Redis的PING请求，twemproxy直接回复PONG，无需转发到后端server。

无论是client connection还是server connection，都有两个队列:

* incoming queue: 存储待处理的请求
* outstanding queue: 存储等待处理完成的请求

接来来看req_forward()。
req_forward()将上述解析出的请求放入client connection的outstanding queue中，
```c
c_conn->enqueue_outq(ctx, c_conn, msg);
```
然后调用server_pool_conn()来选取一个server connection,
```c
kpos = array_get(msg->keys, 0);
key = kpos->start;
keylen = (uint32_t)(kpos->end - kpos->start);

s_conn = server_pool_conn(ctx, c_conn->owner, key, keylen);
```
server_pool_conn()函数会调用到server_pool_idx(), 该函数会根据配置的分发模式来选取server:
```c
switch (pool->dist_type) {
case DIST_KETAMA:
    hash = server_pool_hash(pool, key, keylen);
    idx = ketama_dispatch(pool->continuum, pool->ncontinuum, hash);
    break;

case DIST_MODULA:
    hash = server_pool_hash(pool, key, keylen);
    idx = modula_dispatch(pool->continuum, pool->ncontinuum, hash);
    break;

case DIST_RANDOM:
    idx = random_dispatch(pool->continuum, pool->ncontinuum, 0);
    break;

default:
    NOT_REACHED();
    return 0;
}
```

之后，将解析后的请求同时也放入server connection的incoming queue中，并触发server connection的写事件。
```c
s_conn->enqueue_inq(ctx, s_conn, msg);
```
server connection写事件的回调函数msg_send()会将incoming queue中的请求发送到选定的后端server，发送完成后调用req_send_done():
```c
/* dequeue the message (request) from server inq */
conn->dequeue_inq(ctx, conn, msg);

/*
 * noreply request instructs the server not to send any response. So,
 * enqueue message (request) in server outq, if response is expected.
 * Otherwise, free the noreply request
 */
if (!msg->noreply) {
    conn->enqueue_outq(ctx, conn, msg);
} else {
    req_put(msg);
}
```
该函数会将请求从server connection的incoming queue中取出并放入outstanding queue中。
至此，请求转发完成。

请求的响应到达后，server connection的读事件被触发，调用msg_recv()完成数据读取及解析。解析到完整响应后，rsp_recv_done()被调用。
rsp_recv_done()简化逻辑与req_recv_done很相似:
```c
if (rsp_filter(ctx, conn, msg)) {
    return;
}

rsp_forward(ctx, conn, msg);
```
rsp_filter判断响应是否无需返回给客户端。比如，Redis的AUTH请求是由twemproxy自己发出的，这样的请求swallow被置为1，无需返回响应给客户端:
```c
if (pmsg->swallow) {
    conn->swallow_msg(conn, pmsg, msg);

    conn->dequeue_outq(ctx, conn, pmsg);
    pmsg->done = 1;

    ...

    rsp_put(msg);
    req_put(pmsg);
    return true;
}
```

rsp_forward()取出outstanding queue的第一个请求，它即为当前响应对应的请求，将二者建立对应关系, 并触发client connection的写事件。
```c
pmsg = TAILQ_FIRST(&s_conn->omsg_q);
ASSERT(pmsg != NULL && pmsg->peer == NULL);
ASSERT(pmsg->request && !pmsg->done);

s_conn->dequeue_outq(ctx, s_conn, pmsg);
pmsg->done = 1;

/* establish msg <-> pmsg (response <-> request) link */
pmsg->peer = msg;
msg->peer = pmsg;
```
client connection的写事件回调将响应发送给客户端，发送完之后调用rsp_send_done(), 该函数将请求从client connection的outstanding queue队列中取出。
```c
conn->dequeue_outq(ctx, conn, pmsg);
```
至此，一个请求处理过程就结束了。

对于Redis的MGET之类的请求，还涉及将原始请求分拆成对多个server的多个请求，并将多个响应组合成一个响应回复给客户端。本文对此不详述。
