---
layout: post
title: "NGINX变量机制分析"
date: 2014-12-03 10:29:42 +0800
comments: true
categories: NGINX
---
NGINX实现了一套变量机制，使得配置动态内容非常灵活。比如我们可以使用“proxy_cache_key”指令根据我们的需求灵活地设置Cache的KEY。变量机制也为各模块合作提供了一个桥梁，使得模块间配合完成功能更加方便。比如”proxy_pass”指令支持将回源地址设置为变量，我们可以在另一个模块中依据条件对该变量赋值，从而非常简便地完成基于各种策略选择上游地址的功能。

变量机制相关的内部数据结构主要有两种, ngx_http_variable_t和ngx_http_variable_value_t，分别代表变量本身和变量值。

ngx_http_variable_t结构如下：
```c
struct ngx_http_variable_s {
    ngx_str_t                     name;   /* must be first to build the hash */
    ngx_http_set_variable_pt      set_handler;
    ngx_http_get_variable_pt      get_handler;
    uintptr_t                     data;
    ngx_uint_t                    flags;
    ngx_uint_t                    index;
};
```
各成员意义如下:

* name: 变量名称
* set_handler: 赋值函数，主要用于”set”指令，处理请求时执行set指令时调用
* get_handler: 取值函数，当读取该变量时调用该函数得到变量值
* data: 传递给set_handler和get_handler的参数
* flags: 变量属性标志
* index: 变量在cmcf->variables数组中的索引

其中flags取值及意义如下:

* NGX_HTTP_VAR_CHANGEABLE: 变量被添加时如果已有同名变量，则返回该变量，否则会报错认为变量名冲突。
* NGX_HTTP_VAR_NOCACHEABLE: 变量的值不应该被缓存。变量被取值后，变量值的no_cacheable被置为1。
* NGX_HTTP_VAR_INDEXED：表示变量被索引，存储在cmcf->variables数组，这样的变量可以通过索引值直接找到。
* NGX_HTTP_VAR_NOHASH: 不会将该变量存储在cmcf->variables_hash哈希表。

ngx_http_variable_value_t结构如下：
```c
typedef ngx_variable_value_t  ngx_http_variable_value_t;
```
```c
typedef struct {
    unsigned    len:28;

    unsigned    valid:1;
    unsigned    no_cacheable:1;
    unsigned    not_found:1;
    unsigned    escape:1;

    u_char     *data;
} ngx_variable_value_t;
```
各成员意义如下：

* len: 变量值数据长度
* valid: 该变量值是否可用
* no_cacheable: 该变量值是否不能缓存
* not_found: 对应变量不存在
* escape: 变量值内容中的特殊字符是否进行了转义
* data：变量值的数据

因为NGINX所有变量存储在ngx_http_core_module的main级结构ngx_http_core_main_conf_t中(以下简写cmcf)，所以变量的作用范围是整个http{}配置。在某个server{}中添加的变量，在另一个server{}同样可以使用。
```c
typedef struct {
    ......
    ngx_hash_t                 variables_hash;
    ngx_array_t                variables;       /* ngx_http_variable_t */
    ......
    ngx_hash_keys_arrays_t    *variables_keys;
    ......
} ngx_http_core_main_conf_t;
```

NGINX变量有以下3种类型:

* 模块内置变量
* 根据配置动态添加的变量
* 内置规则变量

模块内置变量主要在ngx_http_module_t的preconfiguration阶段中添加。如upstream模块的preconfiguration回调为ngx_http_upstream_add_variables().
```c
static ngx_int_t
ngx_http_upstream_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var, *v;

    for (v = ngx_http_upstream_vars; v->name.len; v++) {
        var = ngx_http_add_variable(cf, &v->name, v->flags);
        if (var == NULL) {
            return NGX_ERROR;
        }

        var->get_handler = v->get_handler;
        var->data = v->data;
    }

    return NGX_OK;
}
```
根据配置动态添加的变量一般当解析相应指令时，在指令的解析函数中添加。比如rewrite模块中的set指令可以添加用户定义名称的变量。
```c
v = ngx_http_add_variable(cf, &value[1], NGX_HTTP_VAR_CHANGEABLE);
if (v == NULL) {
    return NGX_CONF_ERROR;
}
```
内置规则变量不需要添加，而是按特定规则来解析。如”http_”, “upstream_http_”, “arg_”, “cookie_”等等一系列变量。
前两种方式都是调用ngx_http_add_variable()来添加变量。ngx_http_add_variable()向cmcf->variable_keys数组中添加变量，并将该变量结构返回。如果指定了NGX_HTTP_VAR_CHANGEABLE标志，那么当检查到同名的变量时，则直接返回该变量。否则报错返回NULL。
如果某一指令需要用到一个变量，则一般在解析该指令配置时会调用ngx_http_get_variable_index()，并将该索引值保存，当处理请求时直接通过索引找到变量。如geo模块中geo指令的解析函数。
```c
geo->index = ngx_http_get_variable_index(cf, &name);
if (geo->index == NGX_ERROR) {
    return NGX_CONF_ERROR;
}
```

ngx_http_get_variable_index()会从cmcf->variables数组中查找变量，查找到则返回该变量的索引。否则在cmcf->variables添加该变量并返回数组索引。当解析完HTTP{}配置后，NGINX会将cmcf->variables_keys中的变量组织到cmcf->variables_hash这个HASH表中。如果变量指令了NGX_HTTP_VAR_NOHASH标志，则该变量不会被添加到cmcf->variables_hash中。如果一个变量既没有添加到cmcf->variables中，也没有添加到cmcf->variables_hash中，那么这个变量就不能被找到，因而会被认为不存在。

NGINX开始处理请求时会在请求结构体ngx_http_request_中创建被索引的变量值的缓存空间。
```c
r->variables = ngx_pcalloc(r->pool, cmcf->variables.nelts
                                    * sizeof(ngx_http_variable_value_t));
```
对于被索引的变量，可以使用ngx_http_get_indexed_variable()或者ngx_http_get_flushed_variable()来求值。这样省去了查找哈希表的消耗。ngx_http_get_indexed_variable()首先检查r->variables[index]变量缓存是否可用。可用则直接返回，否则调用v->get_handler对变量求值，并将结果存储在r->variables[index]中。
```c
ngx_http_variable_value_t *
ngx_http_get_indexed_variable(ngx_http_request_t *r, ngx_uint_t index)
{
    ngx_http_variable_t        *v;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    if (cmcf->variables.nelts <= index) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      "unknown variable index: %d", index);
        return NULL;
    }

    if (r->variables[index].not_found || r->variables[index].valid) {
        return &r->variables[index];
    }

    v = cmcf->variables.elts;

    if (v[index].get_handler(r, &r->variables[index], v[index].data)
        == NGX_OK)
    {
        if (v[index].flags & NGX_HTTP_VAR_NOCACHEABLE) {
            r->variables[index].no_cacheable = 1;
        }

        return &r->variables[index];
    }

    r->variables[index].valid = 0;
    r->variables[index].not_found = 1;

    return NULL;
}
```
ngx_http_get_flushed_variable()还会对变量值的no_cacheable标志进行检查。如果为0，表示变量值可以cache, 则直接返回已缓存的变量值。否则，将变量值置为不可用，调用ngx_http_get_indexed_variable()对变量求值。
```c
ngx_http_variable_value_t *
ngx_http_get_flushed_variable(ngx_http_request_t *r, ngx_uint_t index)
{
    ngx_http_variable_value_t  *v;

    v = &r->variables[index];

    if (v->valid || v->not_found) {
        if (!v->no_cacheable) {
            return v;
        }

        v->valid = 0;
        v->not_found = 0;
    }

    return ngx_http_get_indexed_variable(r, index);
}
```
没有索引的变量，可以调用ngx_http_get_variable()完成取值。它会查找cmcf->variables_hash哈希表，找到变量，从相应变量缓存中取值或调用变量的get_handler。
```c
cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

v = ngx_hash_find(&cmcf->variables_hash, key, name->data, name->len);

if (v) {

    if (v->flags & NGX_HTTP_VAR_INDEXED) {
        return ngx_http_get_flushed_variable(r, v->index);

    } else {

        vv = ngx_palloc(r->pool, sizeof(ngx_http_variable_value_t));

        if (vv && v->get_handler(r, vv, v->data) == NGX_OK) {
            return vv;
        }

        return NULL;
    }
}
```
NGINX中rewrite模块实现了set指令，可以给变量赋值。如果另一模块的变量也可以使用set来赋值，则多模块配合完成功能会更加灵活。
set指令的解析函数ngx_http_rewrite_set()代码如下:
```c
static char *
ngx_http_rewrite_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_rewrite_loc_conf_t  *lcf = conf;

    ngx_int_t                            index;
    ngx_str_t                           *value;
    ngx_http_variable_t                 *v;
    ngx_http_script_var_code_t          *vcode;
    ngx_http_script_var_handler_code_t  *vhcode;

    value = cf->args->elts;

    if (value[1].data[0] != '$') {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid variable name "%V"", &value[1]);
        return NGX_CONF_ERROR;
    }

    value[1].len--;
    value[1].data++;

    v = ngx_http_add_variable(cf, &value[1], NGX_HTTP_VAR_CHANGEABLE);
    if (v == NULL) {
        return NGX_CONF_ERROR;
    }

    index = ngx_http_get_variable_index(cf, &value[1]);
    if (index == NGX_ERROR) {
        return NGX_CONF_ERROR;
    }

    if (v->get_handler == NULL
        && ngx_strncasecmp(value[1].data, (u_char *) "http_", 5) != 0
        && ngx_strncasecmp(value[1].data, (u_char *) "sent_http_", 10) != 0
        && ngx_strncasecmp(value[1].data, (u_char *) "upstream_http_", 14) != 0)
    {
        v->get_handler = ngx_http_rewrite_var;
        v->data = index;
    }

    if (ngx_http_rewrite_value(cf, lcf, &value[2]) != NGX_CONF_OK) {
        return NGX_CONF_ERROR;
    }

    if (v->set_handler) {
        vhcode = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                   sizeof(ngx_http_script_var_handler_code_t));
        if (vhcode == NULL) {
            return NGX_CONF_ERROR;
        }

        vhcode->code = ngx_http_script_var_set_handler_code;
        vhcode->handler = v->set_handler;
        vhcode->data = v->data;

        return NGX_CONF_OK;
    }

    vcode = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                       sizeof(ngx_http_script_var_code_t));
    if (vcode == NULL) {
        return NGX_CONF_ERROR;
    }

    vcode->code = ngx_http_script_set_var_code;
    vcode->index = (uintptr_t) index;

    return NGX_CONF_OK;
}
```
它向lcf->codes函数引擎添加rewrite阶段需要执行的函数。如果检测到变量的set_handler存在，则添加ngx_http_script_var_set_handler_code函数，它会调用set_handler。而如果没有set_handler, 则添加ngx_http_script_set_var_code函数，它不会调用set_handler。由于内置变量添加一般是在preconfiguration中完成，因而解析set指令时，变量的set_handler存在，可以正常处理。而根据配置动态添加的变量如果解析出现在set指令后，set指令先被解析，此时变量的set_handler为空，此时添加的函数为ngx_http_script_set_var_code，v->set_handler不会得到调用。因此个人感觉添加函数引擎的逻辑应该放到postconfiguration中处理。此时和变量的各成员值都已被正常赋值。因而可以更方便地让根据配置动态添加的变量也可以和set指令轻松结合。
