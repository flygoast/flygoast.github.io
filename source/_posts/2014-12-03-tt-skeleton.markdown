---
layout: post
title: "Abstract Database和Skeleton Database"
date: 2014-12-03 11:30:50 +0800
comments: true
categories: TokyoCabinet,TokyoTyrant
---
TokyoCabinet(TC)提供了6种不同结构的数据库，包括:

* (MDB) on-memory hash database
* (NDB) on-memory tree database
* (HDB) hash database
* (BDB) B+ tree database
* (FDB) fixed-length database
* (TDB) table database

每种数据库都有各自一套API来进行各种操作。

为了简化使用，TC还提供了一套通用的API来操作以上所有类型数据库，叫做Abstract Database API.

Abstract Database API通过数据库名称来区分各类型数据库:

* "*"          on-memory hash database
* "+"          on-memory tree database
* ".tch"       hash database
* ".tcb"       B+ tree database
* ".tcf"       fixed-length database
* ".tct"       table database

不仅如此，TC更进一步进行了抽象，在Abstract Database中还提供了一种Skeleton Database。
通过实现Skeleton Database指定的API，可以使用自定义的数据库类型。
Skeleton Database API结构体如下：
```c
typedef struct {                        /* type of structure for a extra database skeleton */
  void *opq;                            /* opaque pointer */
  void (*del)(void *);                  /* destructor */
  bool (*open)(void *, const char *);
  bool (*close)(void *);
  bool (*put)(void *, const void *, int, const void *, int);
  bool (*putkeep)(void *, const void *, int, const void *, int);
  bool (*putcat)(void *, const void *, int, const void *, int);
  bool (*out)(void *, const void *, int);
  void *(*get)(void *, const void *, int, int *);
  int (*vsiz)(void *, const void *, int);
  bool (*iterinit)(void *);
  void *(*iternext)(void *, int *);
  TCLIST *(*fwmkeys)(void *, const void *, int, int);
  int (*addint)(void *, const void *, int, int);
  double (*adddouble)(void *, const void *, int, double);
  bool (*sync)(void *);
  bool (*optimize)(void *, const char *);
  bool (*vanish)(void *);
  bool (*copy)(void *, const char *);
  bool (*tranbegin)(void *);
  bool (*trancommit)(void *);
  bool (*tranabort)(void *);
  const char *(*path)(void *);
  uint64_t (*rnum)(void *);
  uint64_t (*size)(void *);
  TCLIST *(*misc)(void *, const char *, const TCLIST *);
  bool (*putproc)(void *, const void *, int, const void *, int, TCPDPROC, void *);
  bool (*foreach)(void *, TCITER, void *);
} ADBSKEL;
```
各成员与其它类型API相应成员意义一致。在开发时，只需实现功能必需的相应函数，忽略其他成员。

使用示例：
```c
ADBSKEL skel;
memset(0, &skel, sizeof(skel));
skel.opq = mydbnew();
skel.del = mydbdel;
skel.open = mydbopen;
skel.close = mydbclose;
...
TCADB *adb = tcadbnew();
tcadbsetskel(adb, &skel);
tcadbopen(adb, "foobarbaz");
```
为了解决多进程共享访问和远程访问TC数据库的不便与繁琐，TC作者开发了一个网络访问层，叫做TokyoTyrant(TT)。它使用TC的Abstract Database API来访问TC数据库。因而内置支持skeleton database扩展。
TT提供了-skel命令行选项来指定skeleton database，启动时它会加载传入的Shared Object(SO)文件，使用SO中定制的数据库实现。
```
ttserver -skel ttskelfoo.so
```
我们可以根据需求实现特定的SO文件，就可以完整利用TT本身已经实现的各种特性，如主备同步，memcache协议支持，HTTP协议支持等。在性能满足需求的情况，这将大大减少开发量。SO文件必须导出一个名字为initialize的函数，TT启动时会从SO文件中查找该函数来初始化skeleton database。
该函数原型为：
```c
bool (*initfunc)(ADBSKEL *);
```
该函数传入一个指向skeleton database的指针。initialize函数中需要将skeleton database定制的数据库操作的API实现赋值到相应函数指针。
由于initialize函数没有参数传递TT本身相关信息，如命令行选项，配置结构等，而TT将一些信息存储在全局变量g_serv指向的TTSERV结构体中，因而SO中可以声明g_serv外部变量来引用。
```c
extern TTSERV  *g_serv;
```
不过较为遗憾的是TTSERV中的信息较少，如有需要的话可以自行扩展。如果SO中逻辑需要依赖命令行选项，可以通过使用启动TT时传入的数据库名来做不同处理。skeleton database的open函数会传入该参数。


具体例子可以参考：
https://github.com/flygoast/ttskeliplist
