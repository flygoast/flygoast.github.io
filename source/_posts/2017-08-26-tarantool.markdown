---
layout: post
title: "Tarantool介绍"
date: 2017-08-26 15:26:28 +0800
comments: true
categories: Tarantool
---
[Tarantool](http://tarantool.org)是俄罗斯最大的互联网公司Mail.RU开发的自带DBMS的Lua应用服务器。从作用上看，它类似于带有持久化数据存储模块的`node.js`。

现代WEB应用程序一般由专门的WEB服务器(如NGINX或APACHE)接入用户请求，然后转发给应用服务器进行业务逻辑处理。应用服务器需要从存储服务中获取数据并加工组织为特定格式的响应，发送给前端的WEB服务器，再由WEB服务器返回给用户。存储服务根据承载介质可分为磁盘存储和内存存储。最常使用的磁盘存储是关系型数据库，如MySQL、PostgreSQL。应用程序对数据库的要求主要是ACID特性。一般情况下，数据库的性能往往满足不了现代应用程序的性能要求。这种情况下，需要在应用程序中引入内存型存储做为cache来满足`RPS(Request Per Second`)要求，如Memcached, Redis等。典型的WEB应用架构如图:

{% img /images/2017-08-26/1.png %}

<!--more-->

这种架构存在一个问题:为了保证持久存储和cache之间的数据一致性，应用程序开发者需要自己处理DBMS和cache之间的同步逻辑。Tarantool将应用服务器，持久存储和缓存这三个组件功能集成在一起，使得开发者精力集中在业务逻辑开发实现，而不用关心持久存储与cache的同步，同时消除应用服务器与存储之间的网络访问。整体架构如图:

{% img /images/2017-08-26/2.png %}

Tarantool包含了一系列模块，其中box模块实现了DBMS功能，http模块可以令客户端以HTTP协议访问tarantool，具体模块信息可以参考: 
https://tarantool.org/en/doc/1.7/reference/index.html

Tarantool的DBMS数据模型是没有模式的(schema-less)，而且它没有关系数据库中的数据库概念，最外层的容器对象叫做`space`, 对应关系数据库中的`table`, `space`中的数据条目叫做`tuple`，一般表示为Lua `table`结构。Tarantool支持二级索引，可以很方便的检索数据。支持的操作为:`insert`, `update`, `upsert`, `replace`, `delete`以及`select`。整体概念和操作上类似于MongoDB。Tarantool将所有对数据库的更新操作都记录到`WAL(Write ahead log)`文件中。若异常发生服务重启后，tarantool可以根据`WAL`文件重建数据。除了`WAL`文件，tarantool还可以生成快照文件，将所有数据内容dump到快照文件中。快照文件生成后将自动删除之前的WAL文件。持久化逻辑与Redis的`AOF`和`RDB`非常类似。

下面举例来说明tarantool作为DBMS的使用。
我们创建`test.lua`文件，内容如下:

```lua
box.cfg {
    listen = 3301,
    logger ="tarantool.log",
    log_level = 5,
    logger_nonblock=true,
    background = true,
    pid_file = "tarantool.pid",
    wal_mode = "none",
}

local function create_space()
    box.schema.user.create('demo', {password='123456'})
    box.schema.user.grant('demo', 'read,write,execute', 'universe')
    box.schema.space.create('hosts')
    box.space.hosts:create_index('primary', {type = 'hash', parts = {1, 'STR'}})
end

box.once('schema', create_space)
```

使用tarantool执行`test.lua`:

```bash
tarantool test.lua
```

这样tarantool作为DBMS监听端口`3301`等待用户请求。

Tarantool官方将客户端驱动叫做`connector`, 支持的语言和框架可以参考:
https://tarantool.org/en/download/connectors.html

这里我们使用Python connector来编写客户端脚本。
创建文件client.py，内容如下:

```python
#!/usr/bin/python
from tarantool import Connection

c = Connection("127.0.0.1", 3301, "demo", "123456")
result = c.insert("hosts", ("1.1.1.1", "hello 1"))
print "insert 1.1.1.1"
print result

result = c.insert("hosts", ("2.2.2.2", 22222, "hello 2"))
print "insert 2.2.2.2"
print result

result = c.select("hosts", "1.1.1.1")
print "select 1.1.1.1"
print result
```

执行后结果如下:
```plain
[root@localhost tarantool]# python client.py
insert 1.1.1.1
- [1.1.1.1, hello 1]

insert 2.2.2.2
- [2.2.2.2, 22222, hello 2]

select 1.1.1.1
- [1.1.1.1, hello 1]

[root@localhost tarantool]#
```

上述例子演示了`insert`和`select`操作，Tarantool原生协议除了支持上面提到的DBMS的几种操作外，还支持`call`和`eval`两种操作。`call`操作用来调用Lua存储过程，`eval`操作用来传入Lua代码片段并执行。我们可以将业务逻辑实现在Lua存储过程中，在客户端使用`call`操作来调用相应的Lua存储过程。

下面举例说明`call`的使用。
我们在上述的`test.lua`中添加两个Lua函数，内容如下:

```lua
function hello(name)
    return "hello " .. name
end

function get(key)
    return box.space.hosts:select(key)
end
```

修改客户端脚本内容为:

```python
#!/usr/bin/python
from tarantool import Connection

c = Connection("127.0.0.1", 3301, "demo", "123456")
result = c.insert("hosts", ("1.1.1.1", "hello 1"))
print "insert 1.1.1.1"
print result

result = c.call('hello', "dummy")
print "hello called"
print result

result = c.call('get', "1.1.1.1")
print "get called"
print result
```

执行后结果如下:
```plain
[root@localhost tarantool]# python client.py
insert 1.1.1.1
- [1.1.1.1, hello 1]

hello called
- [hello dummy]

get called
- [1.1.1.1, hello 1]

[root@localhost tarantool]#
```

在上述例子中，为了简单使用Python connector，如果要实现上文中的NGINX直接访问tarantool的架构, 可以使用tarantool官方实现的[NGINX upstream模块](https://github.com/tarantool/nginx_upstream_module), 或者在OpenResty中由[Lua连接tarantool](https://github.com/perusio/lua-resty-tarantool).

此外上述例子中，connector与tarantool通信使用的是tarantool原生协议，这需要额外引入tarantool的connector客户端库。在某些场景下，可能不想额外添加库文件。Tarantool官方实现了http模块和memcached模块，http模块可以令客户端直接以HTTP协议来访问，memcached模块可以令客户端使用memcached协议来访问tarantool。

我们以示例来说明。
创建`http.lua`代码如下:

```lua
#!/usr/bin/env tarantool

box.cfg {
    logger ="tarantool.log",
    background = true,
    pid_file = "tarantool.pid",
    wal_mode = "none",
}

local function create_space()
    box.schema.space.create('hosts')
    box.space.hosts:create_index('primary', {type = 'hash', parts = {1, 'STR'}})
end

box.once('schema', create_space)

local function handler(self)
    local ipaddr = self.peer.host
    box.space.hosts:upsert({ipaddr, 1}, {{'+', 2, 1}})
    return self:render{json = box.space.hosts:select{ipaddr}}
end

local httpd = require('http.server')
local server = httpd.new('127.0.0.1', 3333)
server:route({path = '/'}, handler)
server:start()
```

我们在`box.cfg`函数中取消了`listen`配置，由`http.server`模块配置监听HTTP协议的端口，我们直接用`curl`访问, 结果如下:

```plain
[root@localhost src]# curl http://127.0.0.1:3333/
[["127.0.0.1",1]][root@localhost src]#
[root@localhost src]# curl http://127.0.0.1:3333/
[["127.0.0.1",2]][root@localhost src]#
```
每次访问，访问计数会增加1。

再创建`mc.lua`来演示memcached协议, 内容如下:

```lua
#!/usr/bin/env tarantool

box.cfg {
    logger ="tarantool.log",
    background = true,
    pid_file = "tarantool.pid",
    wal_mode = "none",
}

local memcached = require('memcached')
local instance = memcached.create('my_instance', '0.0.0.0:11211’)
```

我们再以memcached协议来测试:

```plain
[root@localhost tarantool]# printf "set key 0 60 5\r\nvalue\r\n" | nc 127.0.0.1 11211
STORED
[root@localhost tarantool]# printf "get key\r\n" | nc 127.0.0.1 11211
VALUE key 0 5
value
END
[root@localhost tarantool]# printf "set key2 0 60 6\r\nvalue2\r\n" | nc 127.0.0.1 11211
STORED
[root@localhost tarantool]# printf "get key key2\r\n" | nc 127.0.0.1 11211
VALUE key 0 5
value
VALUE key2 0 6
value2
END
[root@localhost tarantool]#
```

现在的Redis版本也支持加载扩展模块和执行Lua脚本，那我们为何还要使用tarantool呢？Tarantool本质上是LuaJIT及一系列Lua模块的组合，而Redis只是简单支持Lua代码片段的解析执行，在运行时支持上tarantool更完善。Redis实现上是单线程的网络IO复用，如果在业务逻辑中有更为复杂的外部网络访问逻需求，则需要将向外部的非阻塞网络访问嵌入原生的IO循环中，并且由于模块能够开放的API有限，可能还需要修改Redis核心代码。Tarantool本身可视为一个框架，并且将网络IO封装为协程实现的同步非阻塞IO，开发的自由度更高，难度也更小。

除此之外，Tarantool支持双主复制结构，Redis只支持主从复制方式。至于性能，tarantool和Redis在不同场景下各有千秋，可以根据场景根据测试结果来选择。

