---
layout: post
title: "NGINX发送响应分析"
date: 2014-11-18 15:09:53 +0800
comments: true
categories: NGINX
---
NGINX中使用`ngx_http_output_filter()`向一个请求发送响应。
```c
ngx_int_t
ngx_http_output_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t          rc;
    ngx_connection_t  *c;

    c = r->connection;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http output filter "%V?%V"", &r->uri, &r->args);

    rc = ngx_http_top_body_filter(r, in);

    if (rc == NGX_ERROR) {
        /* NGX_ERROR may be returned by any filter */
        c->error = 1;
    }

    return rc;
}
```
其中，`r`是请求结构体，`in`为以`ngx_chain_t`结构链接起来的需要发送的内容。`ngx_http_output_filter()`会调用`ngx_http_top_body_filter()`。NGINX采用filter机制对响应进行处理。filter分为`header filter`和`body filter`。NGINX分别将两种filter各构建成一个链表。全局变量`ngx_http_top_header_filter`指向`header filter`链表的头结点，而全局变量`ngx_http_top_body_filter`指向`body filter`链表的头结点。每个filter中用一个变量记录下一个filter，依据情况决定是调用下一个filter，还是直接返回。

body filter链表顺序如下图:
```Raw token data

  +--------------------------+
  |ngx_http_range_body_filter|
  +----------+---------------+
             |
             v
  +----------+---------+
  |ngx_http_copy_filter|
  +----------+---------+
             |
             v
  +----------+-----------------+
  |ngx_http_charset_body_filter|
  +----------+-----------------+
             |
             v
  +----------+-------------+
  |ngx_http_ssi_body_filter|
  +----------+-------------+
             |
             v
  +----------+-------------+
  |ngx_http_postpone_filter|
  +----------+-------------+
             |
             v
  +----------+--------------+
  |ngx_http_gzip_body_filter|
  +----------+--------------+
             |
             v
  +----------+-----------------+
  |ngx_http_chunked_body_filter|
  +----------+-----------------+
             |
             v
  +---------------------+
  |ngx_http_write_filter|
  +---------------------+

```

header filter链表顺序如图:

```Raw token data

  +----------------------------+
  |ngx_http_not_modified_filter|
  +----------+-----------------+
             |
             v
  +----------+------------+
  |ngx_http_headers_filter|
  +----------+------------+
             |
             v
  +----------+-----------+
  |ngx_http_userid_filter|
  +----------+-----------+
             |
             v
  +----------+-------------------+
  |ngx_http_charset_header_filter|
  +----------+-------------------+
             |
             v
  +----------+---------------+
  |ngx_http_ssi_header_filter|
  +----------+---------------+
             |
             v
  +----------+----------------+
  |ngx_http_gzip_header_filter|
  +----------+----------------+
             |
             v
  +----------+-----------------+
  |ngx_http_range_header_filter|
  +----------+-----------------+
             |
             v
  +----------+-------------------+
  |ngx_http_chunked_header_filter|
  +----------+-------------------+
             |
             v
  +----------+-----------+
  |ngx_http_header_filter|
  +----------------------+

```
`ngx_http_write_filter()`是最后一个被调用的`body filter`，它进行真正的网络I/O操作，将响应发送给客户端。实际上, `ngx_http_header_filter()`也是调用`ngx_http_write_filter()`来发送响应中的headers。

`ngx_http_write_filter()`的简化流程如下:

* 检查之前是否有错误发生(c->error被置位)。如果有错误发生，则没有必要再进行网络I/O操作，直接返回NGX_ERROR。
```c
    if (c->error) {
        return NGX_ERROR;
    }
```
* 计算之前没有发送完成的内容大小并检查是否存在特殊标志。为了优化性能，当没有必要立即发送响应且响应内容大小没有达到设置的阀值时，NGINX可以暂时推迟发送该部分响应。参看:`postpone_output`指令。`flush`标志表示需要立即发送响应。`recycled`表示该buffer需要循环使用，因而需要立即发送以释放该buffer被重新使用。`last`标志表示该buffer是响应的最后一部分内容，因而也需要立即发送。

```c
    for (cl = r->out; cl; cl = cl->next) {
        ll = &cl->next;

        ......

        size += ngx_buf_size(cl->buf);

        if (cl->buf->flush || cl->buf->recycled) {
            flush = 1;
        }

        if (cl->buf->last_buf) {
            last = 1;
        }
    }
```
* 计算本次将发送的内容大小，检查是否存在特殊标志，并将内容链接到r->out上。
```c
    for (ln = in; ln; ln = ln->next) {
        cl = ngx_alloc_chain_link(r->pool);
        if (cl == NULL) {
            return NGX_ERROR;
        }

        cl->buf = ln->buf;
        *ll = cl;
        ll = &cl->next;

        ...

        size += ngx_buf_size(cl->buf);

        if (cl->buf->flush || cl->buf->recycled) {
            flush = 1;
        }

        if (cl->buf->last_buf) {
            last = 1;
        }
    }
```
* 根据情况决定是需要真正进行网络I/O操作, 还是直接返回。
```c
    if (!last && !flush && in && size < (off_t) clcf->postpone_output) {
        return NGX_OK;
    }
```
* 真正进行网络I/O操作，发送内容。
```c
    chain = c->send_chain(c, r->out, limit);
```
* 回收发送完成内容的buffer和chain结构, 将没有发送完成的内容存入`r->out`
```c
    for (cl = r->out; cl && cl != chain; /* void */) {
        ln = cl;
        cl = cl->next;
        ngx_free_chain(r->pool, ln);
    }

    r->out = chain;
```
* 根据发送是否完成，返回`NGX_OK`或`NGX_AGAIN`。
```c
    if (chain) {
        c->buffered |= NGX_HTTP_WRITE_BUFFERED;
        return NGX_AGAIN;
    }

    c->buffered &= ~NGX_HTTP_WRITE_BUFFERED;

    if ((c->buffered & NGX_LOWLEVEL_BUFFERED) && r->postponed == NULL) {
        return NGX_AGAIN;
    }

    return NGX_OK;
```
此外，`ngx_http_write_filter()`中也处理了限速发送的逻辑，本文不详述。
