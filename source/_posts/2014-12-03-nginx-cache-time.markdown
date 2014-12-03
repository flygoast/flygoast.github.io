---
layout: post
title: "修改NGINX实现灵活设置文件缓存时间"
date: 2014-12-03 10:16:36 +0800
comments: true
categories: NGINX
---
业务要求Cache服务器能够随时增删允许访问的HOST。而每个HOST有单独的配置，这些配置随时都可能更改。如果单纯采用静态配置文件(nginx.conf)的方式，每次修改都要reload NGINX。如果更改很频繁，会造成服务器上存在大量的NGINX进程，导致服务器负载很高。因而我们将需要随时更改的配置存储于一个独立的配置服务器中。请求处理时，先去配置服务器中获取该请求需要使用的配置，再根据这些配置进行相应的处理。因而，我们可以随时更改配置服务器中的相应内容。
其中一个配置就是文件缓存时间。NGINX中设置文件缓存时间有两种方法：

- 设置proxy_cache_valid指令
- 在上游响应中的添加"Cache-Control" header和"Expires" header

其中上游响应header的优先级更高。当不想使用上游响应header中所设置的缓存时间时，可以使用以下指令来禁用。
```nginx
proxy_ignore_headers X-Accel-Expires;
```
这两种方法都无法满足我们根据动态配置来设置缓存时间的需求。因而我给NGINX添加了一个内置变量"cache_time"来支持灵活地设置缓存时间，并且该种方式具有最高的优先级。这样，可以非常方便地在ngx_lua等第三方模块中根据条件设置不同的缓存时间。

在ngx_http_request_t添加一个cache_time成员，在ngx_http_core_variables数组中添加内置变量"cache_time"，"cache_time"在被赋值时会将值存储在r->cache_time中。
```c
{ ngx_string("cache_time"), ngx_http_variable_request_set_time,
  ngx_http_variable_request_get_time,
  offsetof(ngx_http_request_t, cache_time),
  NGX_HTTP_VAR_CHANGEABLE|NGX_HTTP_VAR_NOCACHEABLE, 0 },
```
```c
static void
ngx_http_variable_request_set_time(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    ngx_str_t  val;
    time_t     valid, *vp;

    val.len = v->len;
    val.data = v->data;

    valid = ngx_parse_time(&val, 1);
    if (valid == (time_t) NGX_ERROR) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid time value "%V"", &val);
        return;
    }

    vp = (time_t *) ((char *) r + data);

    *vp = valid;

    return;
}
```
因为”cache_time”变量需要比上游响应header具有更高的优先级，因而要在上游header处理之后再处理"cache_time"变量。上游响应的header在ngx_http_upstream_process_headers()中进行处理。因而我在upstream模块中添加了一个hook, 该hook在调用完ngx_http_upstream_process_headers()后，开始处理body前被调用。
```c
if (ngx_http_upstream_process_headers(r, u) != NGX_OK) {
    return;
}

if (u->post_headers) {
    rc = u->post_headers(r);
    if (rc != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u, rc);
        return;
    }
}
```
proxy模块在该hook上注册一个函数，这个函数执行时，首先检查上游响应的状态码判断是否需要处理"cache_time"变量。检查通过后，读取"cache_time"变量的值，依据值来进行各种操作。当值为0时，禁用cache.为正值，则将缓存时间修改为该值。当修改cache缓存时间后，将上游响应中的"Cache-Control"和"Expires" header去除，不再发送给下游。
```c
u->post_headers = ngx_http_proxy_post_headers;
```
```c
static ngx_int_t
ngx_http_proxy_post_headers(ngx_http_request_t *r)
{
    ngx_uint_t         i;
    ngx_table_elt_t  **ph;

    if (ngx_http_upstream_check_status(r->upstream->conf->cache_time_valid,
                                       r->upstream->headers_in.status_n)
        == NGX_DECLINED)
    {
        return NGX_OK;
    }

    if (r->cache_time == (time_t) -1) {
        return NGX_OK;
    }

    if (r->cache_time == (time_t) 0) {
        r->upstream->cacheable = 0;
        return NGX_OK;
    }

    r->cache->valid_sec = ngx_time() + r->cache_time;

    r->headers_out.expires->hash = 0;

    ph = r->headers_out.cache_control.elts;
    for (i = 0; i < r->headers_out.cache_control.nelts; i++) {
        ph[i]->hash = 0;
    }

    return NGX_OK;
}
```
