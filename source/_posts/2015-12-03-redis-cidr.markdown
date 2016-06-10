---
layout: post
title: "扩展Redis命令支持CIDR查询"
date: 2015-12-03 20:05:10 +0800
comments: true
categories: Redis
---
我们的NGINX的IP封禁功能基于Redis实现。当只支持单IP封禁时，直接以IP作为KEY，调用”GET”命令，根据Value判断是否需要封禁该IP。若要支持网段封禁，需要取出所有的CIDR段，然后判断IP是否在CIDR范围内。随着CIDR越来越多，从Redis中取出的数据则越来越多，性能消耗越来越大。为了减少数据传输量，则可以将判断逻辑改由Redis来完成。

Redis本身支持Lua脚本的执行，可以由Lua来实现相应逻辑。不过Lua语言本身不支持位运算(5.2之后支持)，需要第三方库支持。所以，我们直接通过修改Redis代码扩展Redis命令来实现该功能。

<!--more-->
Redis中所有命令的相关信息存储在redisCommandTable结构中。我们在其中添加上我们自己的命令，如”setcidr”, “checkcidr”等。
```c
struct redisCommand redisCommandTable[] = {
    {"setcidr",setcidrCommand,-3,"wmF",0,NULL,1,1,1,0,0},
    {"delcidr",delcidrCommand,-3,"wmF",0,NULL,1,1,1,0,0},
    {"checkcidr",checkcidrCommand,3,"rF",0,NULL,1,1,1,0,0},
    {"getcidr",getcidrCommand,2,"rS",0,NULL,1,1,10,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    ...
};
```

其中指定了参数个数，回调函数等相应信息，具体参考源码中该结构上方的注释。

当Redis收到某命令时，会调用指定的回调函数。比如，Redis收到”setcidr”命令时，会调用函数setcidrCommand。

我们来看该函数的实现:
```c
void setcidrCommand(redisClient *c) {
    robj  *cidr, *obj;
    int    j, added = 0;

    cidr = lookupKeyWrite(c->db, c->argv[1]);
    if (cidr == NULL) {
        cidr = setTypeCreate(c->argv[2]);
        dbAdd(c->db, c->argv[1], cidr);
    } else {
        if (cidr->type != REDIS_SET) {
            addReply(c, shared.wrongtypeerr);
            return;
        }
    }

    for (j = 2; j < c->argc; j++) {
        obj = createRawStringObject(NULL, sizeof(in_cidr_t)
                                          + sdslen(c->argv[j]->ptr));
        if (getCidrFromObject(c->argv[j], (in_cidr_t *) obj->ptr) != REDIS_OK) {
            decrRefCount(obj);
            continue;
        }
        memcpy((char *)obj->ptr + sizeof(in_cidr_t),
               c->argv[j]->ptr, sdslen(c->argv[j]->ptr));

        decrRefCount(c->argv[j]);
        c->argv[j] = obj;

        if (setTypeAdd(cidr, c->argv[j])) {
            added++;
        }
    }

    if (added) {
        signalModifiedKey(c->db, c->argv[1]);
        notifyKeyspaceEvent(REDIS_NOTIFY_SET, "setcidr", c->argv[1], c->db->id);
    }
    server.dirty += added;
    addReplyLongLong(c, added);
}
```

我们直接使用用Redis自身的SET类型来存储CIDR。根据参数KEY查找相应的SET。若没有，则调用setTypeCreate来创建。我们使用RawString对象存储CIDR数据，根据参数依次调用createRawStringObject来创建RawString对象，存在SET中。

CIDR数据的结构为:
```c
typedef struct {
    in_addr_t  addr;
    in_addr_t  mask;
} in_cidr_t;
```

当调用checkCIDR命令时，会调用checkcidrCommand函数。
```c
void checkcidrCommand(redisClient *c) {
    robj             *cidr, *ele;
    setTypeIterator  *si;
    in_addr_t         addr;
    in_cidr_t        *ic;
    int               found;

    if ((cidr = lookupKeyReadOrReply(c,c->argv[1],shared.czero)) == NULL) {
        return;
    }

    if (checkType(c, cidr, REDIS_SET)) {
        return;
    }

    addr = inet_addr2((char *) c->argv[2]->ptr, sdslen(c->argv[2]->ptr));
    if (addr == INADDR_NONE) {
        addReply(c, shared.czero);
        return;
    }

    found = 0;
    si = setTypeInitIterator(cidr);
    while ((ele = setTypeNextObject(si)) != NULL) {
        ic = (in_cidr_t *) ele->ptr;

        if ((addr & ic->mask) == ic->addr) {
            found = 1;
            decrRefCount(ele);
            break;
        }

        decrRefCount(ele);
    }
    setTypeReleaseIterator(si);

    if (found) {
        addReply(c, shared.cone);
    } else {
        addReply(c, shared.czero);
    }
}
```

函数中首先根据KEY获取存储CIDR的SET，然后依次进行检查IP是否在CIDR段内，分别返回0或1。

对Redis进行扩展非常简单，这种方法可以实现很多需求。不过由于Redis为单线程，功能实现代码里不能阻塞，否则会影响Redis本身性能。
