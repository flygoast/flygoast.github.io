---
layout: post
title: "ngx_http_limit_req_module分析"
date: 2015-09-08 15:34:36 +0800
comments: true
categories: NGINX 
---
ngx_http_limit_req_module用于依据指定的KEY来限制请求处理速度。比如，用来限制单个IP的访问频率。

文档地址: http://nginx.org/en/docs/http/ngx_http_limit_req_module.html

下面来看具体实现:

NGINX模块从逻辑中主要有两部分：

* 配置解析: 将配置文件的内容解析到各配置结构中
* 请求处理: 在当前请求上完成模块逻辑处理

首先看配置解析部分:

limit_req模块的ngx_http_module_t结构指定了配置解析的HOOK函数:
```c
static ngx_http_module_t  ngx_http_limit_req_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_limit_req_init,               /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_limit_req_create_conf,        /* create location configuration */
    ngx_http_limit_req_merge_conf          /* merge location configuration */
};
```
limit_req模块只使用location级别的配置，只注册了location级别配置的create和merge函数。

ngx_http_limit_req_create_conf()函数简单地从配置内存池中分配一个ngx_http_limit_req_conf_t结构。
```c
typedef struct {
    ngx_array_t                  limits;
    ngx_uint_t                   limit_log_level;
    ngx_uint_t                   delay_log_level;
    ngx_uint_t                   status_code;
} ngx_http_limit_req_conf_t;
```
该结构用于保存一个location{}配置块中的limit_req模块配置。location{}配置块下允许使用多个limit_req指令。如:
```nginx
location /dummy {
    limit_req zone=one burst=5 nodelay;
    limit_req zone=two burst=10;
    ...
}
```
limits数组的每个元素为ngx_http_limit_req_limit_t结构，每条limit_req指令配置都保存到该结构中。
```c
typedef struct {
    ngx_shm_zone_t              *shm_zone;
    /* integer value, 1 corresponds to 0.001 r/s */
    ngx_uint_t                   burst;
    ngx_uint_t                   nodelay; /* unsigned  nodelay:1 */
} ngx_http_limit_req_limit_t;
```
下面看模块limit_req_zone和limit_req指令的解析:
```c
{ ngx_string("limit_req_zone"),
  NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE3,
  ngx_http_limit_req_zone,
  0,
  0,
  NULL },

{ ngx_string("limit_req"),
  NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE123,
  ngx_http_limit_req,
  NGX_HTTP_LOC_CONF_OFFSET,
  0,
  NULL },
```
limit_req_zone指令处理函数为ngx_http_limit_req_zone函数。

首先，函数从配置内存池分配一个ngx_http_limit_req_ctx_t结构。
```c
ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_limit_req_ctx_t));
if (ctx == NULL) {
    return NGX_CONF_ERROR;
}
```
每个ngx_http_limit_req_ctx_t保存limit_req_zone指令的配置内容:
```c
typedef struct {
    ngx_http_limit_req_shctx_t  *sh;
    ngx_slab_pool_t             *shpool;
    /* integer value, 1 corresponds to 0.001 r/s */
    ngx_uint_t                   rate;
    ngx_http_complex_value_t     key;
    ngx_http_limit_req_node_t   *node;
} ngx_http_limit_req_ctx_t;
```
接着，预编译该ZONE所指定的KEY，将编译结果保存key成员变量中，在请求处理阶段模块根据key成员变量得到KEY的值。
```c
ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));

ccv.cf = cf;
ccv.value = &value[1];
ccv.complex_value = &ctx->key;

if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
    return NGX_CONF_ERROR;
}
```
接下来，解析各参数的值，如共享内存区的名称和大小，限制的访问频率。为了避免计算时使用浮点数而提高处理效率，代码中将1r/s对应的rate值设为1000。
```c
ctx->rate = rate * 1000 / scale;
```
再接着，添加共享内存信息:
```c
shm_zone = ngx_shared_memory_add(cf, &name, size,
                                 &ngx_http_limit_req_module);
...
shm_zone->init = ngx_http_limit_req_init_zone;
shm_zone->data = ctx;
```
NGINX此时并不会分配共享内存，只是将信息保存下来，在所有配置解析完后再统一进行分配。NGINX分配完共享内存后，调用设置的ngx_http_limit_req_init_zone来初始化该共享内存区域。
```c
ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;
...
ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_limit_req_shctx_t));
if (ctx->sh == NULL) {
    return NGX_ERROR;
}

ctx->shpool->data = ctx->sh;

ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                ngx_http_limit_req_rbtree_insert_value);

ngx_queue_init(&ctx->sh->queue);
```
ngx_http_limit_req_init_zone函数在共享内存区创建了一个RBTREE和一个QUEUE结构，RBTREE NODE用于保存检查命中各KEY的请求是否应限速所需的信息，QUEUE结构用于回收长期没有访问的节点内存。

NGINX解析完配置后，调用注册在postconfiguration阶段的ngx_http_limit_req_init函数。
```c
static ngx_int_t
ngx_http_limit_req_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_limit_req_handler;

    return NGX_OK;
}
```
该函数在NGINX请求处理的PREACCESS阶段注册了处理函数ngx_http_limit_req_handler。

下面来看请求阶段的处理。

当请求到达时，NGINX按阶段执行到ngx_http_limit_req_handler函数。

handler函数首先判断是否已经进行过limit_req检查。若已进行过，则直接跳到下一个阶段，保证每个请求只由limit_req模块处理一次。
```c
if (r->main->limit_req_set) {
    return NGX_DECLINED;
}
```

接着依据当前请求所属location{}配置中的所有limit_req策略依次调用ngx_http_limit_req_lookup进行检查:
```c
for (n = 0; n < lrcf->limits.nelts; n++) {
    limit = &limits[n];
    ctx = limit->shm_zone->data;

    if (ngx_http_complex_value(r, &ctx->key, &key) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    ...

    hash = ngx_crc32_short(key.data, key.len);
    ngx_shmtx_lock(&ctx->shpool->mutex);
    rc = ngx_http_limit_req_lookup(limit, hash, &key, &excess,
                                   (n == lrcf->limits.nelts - 1));

    ngx_shmtx_unlock(&ctx->shpool->mutex);
    ...
    if (rc != NGX_AGAIN) {
        break;
    }
}
```
ngx_http_limit_req_lookup()首先从RBTREE中查找当前KEY所属节点。
若查找到节点，则把当前节点移到队列首端，保证不会在近期被回收内存。
```c
ngx_queue_remove(&lr->queue);
ngx_queue_insert_head(&ctx->sh->queue, &lr->queue);
```
然后，计算该请求是否超出了限制频率:
```c
ms = (ngx_msec_int_t) (now - lr->last);
excess = lr->excess - ctx->rate * ngx_abs(ms) / 1000 + 1000;
```
这里用的是”leaky bucket”算法。lr->excess表示上次处理后剩余的请求数，ms为现在距上次处理请求的时间，最后的”1000”为当前请求（一个请求为1000），excess为按限定速率进行处理到现在应该剩余的请求数。如果excess超过了设定的暴发阈值,则函数直接返回NGX_BUSY。这会让NGINX直接以limit_req_status_code设定的状态码结束请求（默认值为503）。若没有超过暴发阈值，若当前检查的limit_req规则是LOCATION所配置的最后一个，则将excess和当前时间更新到NODE中，返回NGX_OK, 否则，返回NGX_AGAIN，表示通过了该条limit_req策略，应该继续检查下一条limit_req策略。

若没有找到节点，则创建新的节点插入到RBTREE中。同样的，若是最后一个策略返回NGX_OK, 否则返回NGX_AGAIN。

ngx_http_limit_req_lookup()返回值总结如下:

* NGX_ERROR: 分配内存错误
* NGX_AGAIN: 通过了一个limit_req策略，需要检查下一个策略
* NGX_OK: 通过了所有limit_req策略
* NGX_BUSY: 超过了设置的最大请求频率，直接以状态码结束请求

当通过所有策略后，handler调用ngx_http_limit_req_account检测是否需要delay请求。
```c
delay = ngx_http_limit_req_account(limits, n, &excess, &limit);
```
delay为处理完当前所有剩余请求所需的时间:
```c
tp = ngx_timeofday();

now = (ngx_msec_t) (tp->sec * 1000 + tp->msec);
ms = (ngx_msec_int_t) (now - lr->last);

excess = lr->excess - ctx->rate * ngx_abs(ms) / 1000 + 1000;

if (excess < 0) {
    excess = 0;
}

...

delay = excess * 1000 / ctx->rate;
```
若需要delay请求，则添加一个NGINX TIMER来延迟请求处理。
```c
r->read_event_handler = ngx_http_test_reading;
r->write_event_handler = ngx_http_limit_req_delay;
ngx_add_timer(r->connection->write, delay);
```
至此，limit_req的逻辑处理完成。

limit_req模块在使用NGINX CORE的rbtree代码有些繁琐，比如ngx_http_limit_req_node_t结构用于保存”leaky bucket”算法所需的数据。limit_req模块需要将这些数据附在ngx_rbtree_node_t结构之后，从而复用rbtree操作的相关函数。limit_req模块在该结构的开始用了两个成员变量来标识来ngx_rbtree_node_t的拼接点。两结构定义如下:
```c
struct ngx_rbtree_node_s {
    ngx_rbtree_key_t       key;
    ngx_rbtree_node_t     *left;
    ngx_rbtree_node_t     *right;
    ngx_rbtree_node_t     *parent;
    u_char                 color;
    u_char                 data;
};

typedef struct {
    u_char                       color;
    u_char                       dummy;
    u_short                      len;
    ngx_queue_t                  queue;
    ngx_msec_t                   last;
    /* integer value, 1 corresponds to 0.001 r/s */
    ngx_uint_t                   excess;
    ngx_uint_t                   count;
    u_char                       data[1];
} ngx_http_limit_req_node_t;
```

NODE大小这样计算:
```c
size = offsetof(ngx_rbtree_node_t, color)
       + offsetof(ngx_http_limit_req_node_t, data)
       + key->len;
```
这样理解起来不够直观。可以简单的定义结构为:
```c
typedef struct {
    ngx_rbtree_node_t            node;
    u_short                      len;
    ngx_queue_t                  queue;
    ngx_msec_t                   last;
    /* integer value, 1 corresponds to 0.001 r/s */
    ngx_uint_t                   excess;
    ngx_uint_t                   count;
    u_char                       data[1];
} ngx_http_limit_req_node_t;
```
这样计算NODE大小则为:
```c
size = offsetof(ngx_http_limit_req_node_t, data)
       + key->len;
```
另外对于我们特定的需求，limit_req_zone限定的访问频率有两个不方便之处:

* 不能针对不同的KEY，限定不同的访问频率
* 不能实时动态更改, 只能通过修改配置RELOAD NGINX来生效

针对上述两个不方便之处，可以给limit_req_zone添加一个RATE变量表示需要对该请求的限制频率，在处理请求时动态获取到该值。而该变量可以在另外的模块中根据更情况来设置，比如，可以由NGX_LUA模块从REDIS等动态存储中获取相关信息而设置。

但这种方式有两点需注意:

* 针对同一个KEY的请求要保证频率值不变，否则失去了频率的意义
* RATE变量的设置要在PREACCESS阶段前，比如在SERVER REWRITE阶段

文中代码为nginx-1.8.0。
