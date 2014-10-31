---
layout: post
title: "NGINX按目录清除缓存"
date: 2014-11-01 00:40:46 +0800
comments: true
categories: NGINX
---

NGINX只在商业版中支持proxy_cache_purge指令清除缓存，开源的ngx_cache_purge模块只支持单一key的缓存清除。为了实现按目录清除缓存只能自己开发。

NGINX作为Cache服务器时将资源内容以文件形式进行缓存，缓存元信息存储于共享内存中，组织成一棵红黑树。红黑树中的每个节点代表一个Cache元信息。NGINX将Cache Key的HASH值作为红黑树节点的KEY。内容缓存文件以该HASH值作为文件名存储在磁盘上。

NGINX的处理流程简化描述是这样的：当请求到达时，根据Cache Key的HASH值在红黑树中进行查找。如果找到，并查看相关信息，如果Cache可用，返回相应的Cache文件。否则，则回源抓取。

因为元信息是以Cache Key的HASH值作为Key存储的，因而红黑树中并不能保留Cache Key中有层级关系. 如"/uri/foo"和"/uri/bar"在元信息红黑树中完全没有关系。要实现按照目录清除缓存，需要将Cache Key中层次关系存储起来。

我选择的方案是在共享内存中建立一棵目录树来存储层级关系。将Cache Key类比于文件系统中的路径, 每级路径存储为树中的一个节点。当需要清除某一目录下的所有缓存时，将该节点子树的中的所有缓存清除即可。

目录树采用Child-Sibling链表实现。节点结构定义如下:
```
typedef struct ngx_http_file_cache_path_node_s ngx_http_file_cache_path_node_t;

struct ngx_http_file_cache_path_node_s {
    ngx_hlist_node_t                  node;
    ngx_http_file_cache_path_node_t  *parent;
    ngx_http_file_cache_path_node_t  *child;
    ngx_http_file_cache_path_node_t  *next;
    ngx_http_file_cache_path_node_t  *prev;
    ngx_http_file_cache_node_t       *rb_node;
    unsigned                          count:32;
    unsigned                          depth:8;
    unsigned                          isdir:1;
                                      /* 23 unused bits */
    ngx_uint_t                        len;
    u_char                            data[0];
};
```

以纵向表示孩子关系，横向表示兄弟关系，则Cache key "/uri/foo"则生成如下结构：
```
+-----+
| uri |
+--+--+
   |
+--+--+
| foo |
+-----+
```

再缓存Cache Key "/uri/bar"之后则结构为:
```
+-----+
| uri |
+--+--+
   |
+--+--+    +-----+
| bar +----+ foo |
+-----+    +-----+
```

其中node字段用于将节点加入HASH表中。考虑到如果一个目录下子目录或文件太多，则遍历兄弟链表则非常耗时。因而引入一个HASH表，将所有树结点以路径节点名字作为key加入HASH表。当子节点少，直接遍历。子节点过多时，则从HASH中进行查找。由于不同位置的路径节点会重名，如"/uri/foo/dummy"和"/uri/bar/dummy"两个名字为"dummy"的路径节点分别指向不同的Cache。因而在HASH表中查找时需要考虑路径节点的父节点。
如，判断一个中间目录路径节点时代码：
```
if (pos->isdir
    && pos->parent == parent
    && pos->len == path[i].len
    && ngx_strncmp(pos->data, path[i].data, path[i].len)
       == 0)
{
   ......
}
```

非叶子节点代表路径。叶子节点可能为目录或是缓存文件。代表缓存文件的叶子节点中则保存该Cache的元信息节点的地址。同时在元信息中也添加上该叶子节点的地址。因而当找到代表要清除的目录节点时，遍历子树便可以找到所有缓存信息的元信息，对元信息进行相关操作，完成清除操作。
当删除Cache文件时，将Cache key的路径节点也一并删除，回收内存。
