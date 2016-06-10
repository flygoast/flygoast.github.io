---
layout: post
title: "ngx_http_limit_conn_module模块分析"
date: 2015-09-13 18:58:48 +0800
comments: true
categories: NGINX 
---
ngx_http_limit_conn_module模块用来限制某个KEY的并发连接数。它的实现与ngx_http_limit_req_module模块类似，整体逻辑和实现更为简单。LimitConn模块也将某个KEY的信息存储在共享内存RBTREE中的节点中, 但不需要QUEUE结构。LimitConn模块只需要在节点中记录当前的连接数信息:
```c
typedef struct {
    u_char                     color;
    u_char                     len;
    u_short                    conn;
    u_char                     data[1];
} ngx_http_limit_conn_node_t;
```

<!--more-->
handler也注册在PREACCESS阶段。因为连接只有被NGINX处理并且读取完HTTP请求的Header后，才开始进行阶段处理，所以只有达到这个阶段的连接才会被计数。

handler的整体逻辑简化如下:
```c
static ngx_int_t
ngx_http_limit_conn_handler(ngx_http_request_t *r)
{
    ...

    if (r->main->limit_conn_set) {
        return NGX_DECLINED;
    }

    lccf = ngx_http_get_module_loc_conf(r, ngx_http_limit_conn_module);
    limits = lccf->limits.elts;

    for (i = 0; i < lccf->limits.nelts; i++) {
        //处理每一条limit_conn策略
    }
    return NGX_DECLINED;
}
```
handler中依次处理配置的每一条limit_conn策略。任何一条策略匹配，就以503或用limit_conn_status_code指令定义的状态码结束请求。

处理limit_conn策略的过程如下:
首先根据KEY在RBTREE中查找节点:
```c
node = ngx_http_limit_conn_lookup(ctx->rbtree, &key, hash);
```
若没有找到节点，说明该请求是这个KEY的第一个请求，这时创建一个节点，将连接数初始化为1，添加到RBTREE中。
```c
n = offsetof(ngx_rbtree_node_t, color)
    + offsetof(ngx_http_limit_conn_node_t, data)
    + key.len;

node = ngx_slab_alloc_locked(shpool, n);

if (node == NULL) {
    ngx_shmtx_unlock(&shpool->mutex);
    ngx_http_limit_conn_cleanup_all(r->pool);
    return lccf->status_code;
}

lc = (ngx_http_limit_conn_node_t *) &node->color;

node->key = hash;
lc->len = (u_char) key.len;
lc->conn = 1;
ngx_memcpy(lc->data, key.data, key.len);

ngx_rbtree_insert(ctx->rbtree, node);
```
若找到节点，则判断是否超过了限定的连接数，超过则结束请求，否则将节点中存储的连接数加1。
```c
lc = (ngx_http_limit_conn_node_t *) &node->color;

if ((ngx_uint_t) lc->conn >= limits[i].conn) {

    ngx_shmtx_unlock(&shpool->mutex);

    ngx_log_error(lccf->log_level, r->connection->log, 0,
                  "limiting connections by zone \"%V\"",
                  &limits[i].shm_zone->shm.name);

    ngx_http_limit_conn_cleanup_all(r->pool);
    return lccf->status_code;
}

lc->conn++;
```
若请求通过了策略，请求结束时则应该将连接数减1。模块将这逻辑的处理函数注册到请求的内存池销毁阶段：
```c
cln = ngx_pool_cleanup_add(r->pool,
                           sizeof(ngx_http_limit_conn_cleanup_t));
if (cln == NULL) {
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}

cln->handler = ngx_http_limit_conn_cleanup;
lccln = cln->data;

lccln->shm_zone = limits[i].shm_zone;
lccln->node = node;
```
ngx_http_limit_conn_cleanup函数将连接数减1，若连接数减到0，说明已没有连接，销毁节点回收内存。
```c
ngx_http_limit_conn_cleanup(void *data)
{
    ngx_http_limit_conn_cleanup_t  *lccln = data;

    ...

    ctx = lccln->shm_zone->data;
    shpool = (ngx_slab_pool_t *) lccln->shm_zone->shm.addr;
    node = lccln->node;
    lc = (ngx_http_limit_conn_node_t *) &node->color;

    ngx_shmtx_lock(&shpool->mutex);

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, lccln->shm_zone->shm.log, 0,
                   "limit conn cleanup: %08XD %d", node->key, lc->conn);

    lc->conn--;

    if (lc->conn == 0) {
        ngx_rbtree_delete(ctx->rbtree, node);
        ngx_slab_free_locked(shpool, node);
    }

    ngx_shmtx_unlock(&shpool->mutex);
}
```
上述当以503结束请求前，都会调用ngx_http_limit_conn_cleanup_all函数。一个连接会检测多个limit_conn策略，当然请求可能已经在之前的limit_conn策略对应KEY的节点中给增加了连接数。这里立即将这种连接数递减。
```c
static ngx_inline void
ngx_http_limit_conn_cleanup_all(ngx_pool_t *pool)
{
    ngx_pool_cleanup_t  *cln;

    cln = pool->cleanup;

    while (cln && cln->handler == ngx_http_limit_conn_cleanup) {
        ngx_http_limit_conn_cleanup(cln->data);
        cln = cln->next;
    }

    pool->cleanup = cln;
}
```
在实际应用中，为了用户体验，期望用户第一次请求通过后，以后则不再限制，直接放行。可以在第一次请求时，设置特定COOKIE，下一次请求时，验证COOKIE通过则直接放行。用NGX_LUA可以轻松实现LimitConn模块逻辑和上述放行逻辑。
