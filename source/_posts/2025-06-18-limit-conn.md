---
title: 关于nginx中的limit_conn指令
date: 2025-06-18 10:34:45
tags:
- nginx
categories: NGINX 
---
多年前写过介绍`nginx`限制连接模块的文章[<<ngx_http_limit_conn_module模块分析>>](2015/09/13/ngx-http-limit-conn-module/), 最近业务中用到`limit_conn`指令限制请求，重新了解了一下它的用法。

根据[`nginx`文档](https://nginx.org/en/docs/http/ngx_http_limit_conn_module.html#limit_conn), 可以理解主要逻辑是根据`limit_conn_zone`所指定的`key`值计算连接数，当连接数超过`limit_conn`所指定的值时，则返回错误码。

由于限制值是在`limit_conn`指令而不是在`limit_conn_zone`指令中设置的，而`limit_conn`是可以配置多次的，当配置多次相同`zone`的`limit_conn`指令并且限制值不同，那么生效的是哪个限制值呢？

如果在同一个`location`中配置多个相同`zone`的`limit_conn`指令，示例配置为:
```plain
    location /a {
        limit_conn perserver 2;
        limit_conn perserver 5;
        ...
    }
```

<!--more-->

当启动`nginx`会检测到这种情况直接报错:
```plain
nginx: [emerg] "limit_conn" directive is duplicate in /usr/local/nginx/conf/nginx.conf:52
```

查看`nginx`源码，在处理`limit_conn`指令配置时，会对相同`zone`的情况进行检测，当有重复的`zone`, 则直接报错:
```c
static char *
ngx_http_limit_conn(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ......

    for (i = 0; i < lccf->limits.nelts; i++) {
        if (shm_zone == limits[i].shm_zone) {
            return "is duplicate";
        }
    }
```

如果在不同`location`下配置相同`zone`的`limit_conn`指令，如:
```plain
limit_conn_zone $server_name zone=perserver:10m;

server {
    ...
    location /a {
        limit_conn perserver 2;
        ...
    }

    location /b {
        limit_conn perserver 5;
    }
}
```

这种情况下，`nginx`可以正常启动，限制值如何生效呢？

根据`nginx`源码, 配置的限制值存储在`ngx_http_limit_conn_limit_t`结构中，该结构是一个`location`级别的结构，因而对于不同的`location`是各自独立的:
```c
typedef struct {
    ngx_shm_zone_t               *shm_zone;
    ngx_uint_t                    conn;
} ngx_http_limit_conn_limit_t;
```

实际的检测逻辑注册在`PREACCESS`阶段:
```c
static ngx_int_t
ngx_http_limit_conn_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_limit_conn_handler;

    return NGX_OK;
}
```

检测时，会从对应的`location`结构中获取`ngx_http_limit_conn_limit_t`结构，因而使用的是自身`location`中所配置的限制值:

```c
static ngx_int_t
ngx_http_limit_conn_handler(ngx_http_request_t *r)
{
    size_t                          n;
    uint32_t                        hash;
    ngx_str_t                       key;
    ngx_uint_t                      i;
    ngx_rbtree_node_t              *node;
    ngx_pool_cleanup_t             *cln;
    ngx_http_limit_conn_ctx_t      *ctx;
    ngx_http_limit_conn_node_t     *lc;
    ngx_http_limit_conn_conf_t     *lccf;
    ngx_http_limit_conn_limit_t    *limits;
    ngx_http_limit_conn_cleanup_t  *lccln;

    if (r->main->limit_conn_status) {
        return NGX_DECLINED;
    }

    lccf = ngx_http_get_module_loc_conf(r, ngx_http_limit_conn_module);
    limits = lccf->limits.elts;

    for (i = 0; i < lccf->limits.nelts; i++) {
    ......

```

因而, 对于上述的配置示例，并发连接数是基于`zone`中的`key`计算的，因而无论访问`/a`还是`/b`都会增加对应`zone: perserver`的并发连接数，但实际是否超过限制值是每个`location`单独计算的。

比如，当前已经有了一个`/a`和一个`/b`的连接，此时再访问`/a`, 由于`location /a`配置的限制值为`2`, 已达到上限，新连接被限制，返回`503`或指定的错误码，而这时对于`/b`的新连接，由于总连接数小于`5`, 可以正常进行。
