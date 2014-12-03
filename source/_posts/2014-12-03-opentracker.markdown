---
layout: post
title: "Opentracker扩展"
date: 2014-12-03 11:27:11 +0800
comments: true
categories: OPENTRACKER
---
[opentracker](http://erdgeist.org/arts/software/opentracker)是一个开源P2P tracker服务器.之前我们系统中主要使用的是PHP+MYSQL实现的peertracker。随着业务增长，peertracker的性能已经不能满足系统需要。因而我们决定引入性能更好的opentracker。不过, opentracker在功能上并不能完全满足我们的需求，因而我对它进行了一些扩展。

- UDP单播同步数据

opentracker本身支持cluster模式。cluster内各节点之间会同步数据。这样可以通过添加节点提高整体的集群性能。然而，opentracker原生通过UDP多播进行数据同步。我们不具备多播IP，因而开发了单播模式进行同步。实现非常简单，就是依次向cluster内其他节点发送数据。缺点是当节点数较多时，会影响性能。

- 持久化支持

opentracker将torrent和peer信息保存在内存中。当opentracker重启时，所有的torrent和peer信息就都丢失了。这会导致我们的系统一段时间内不能进行正常的P2P传输。因而我扩展了持久化功能。opentracker架构上通过多个不同的线程执行不同的任务。我添加了一个线程，周期性地将内存中的torrent和peer信息保存到磁盘文件中。这个线程很像Redis中进行数据dump的进程。磁盘文件格式定义为ODB(opentracker database)，它主要借鉴自Redis的RDB格式。目前，没有处理IPV6格式，因而只支持IPV4。
我还提供了一个工具支持流式地对odb文件进行处理。

具体用法请参考: `perldoc OdbParser`

格式规范:
```
----------------------------------- # ODB is a binary format. There are no new lines or spaces in the file.
4f 50 45 4e 54 52 41 43 4b 45 52    # Magic String "OPENTRACKER"
30 30 30 33                         # ODB Version Number in ASCII characters. In this case, version = "0001" = 1
-----------------------------------
FE                                  # FE = Opcode that indicates following is a torrent information.
----------------------------------- # Torrent information starts from here.
00 02 2e 33 01 c4 a1 df bb 82
1f 51 0f b5 b6 02 6f 93 6e 9f       # 20 byte info_hash of torrent
-----------------------------------
e2 37 59 01 00 00 00 00             # Last access time in minutes since UNIX epoch, 8 bytes.
                                    # At present, when loading ODB file, just use current clock, ignore this.
-----------------------------------
00 00 00 00 00 00 00 00             # Seeding peer count for current torrent. 8 bytes long integer in little endian.
                                    # NOT used at present.
-----------------------------------
00 00 00 00 00 00 00 00             # Total peer count for current torrent. 8 bytes long integer in little endian.
                                    # NOT used at present.
-----------------------------------
00 00 00 00 00 00 00 00             # Download times of files in current torrent. 8 bytes long integer in little endian.
                                    # NOT used at present.
-----------------------------------
01 00 00 00                         # Peer count in current peer, 4 bytes integer in little endian.
----------------------------------- # Peers information starts from here.
7f 00 00 01                         # 4 bytes ip address in network byte order. In this case, 0x7f000001 = "127.0.0.1"
1b 31                               # 2 bytes port in network byte order. In this case, 0x1b31 = 6961.
80                                  # Flag of peers. SEEDING = 0x80, COMPLETED = 0x40, STOPPED = 0x20, LEECHING = 0x00
00                                  # Reserved. Just set zero.
-----------------------------------
...                                 # Other peers information.
-----------------------------------
FE
-----------------------------------
...                                 # Other torrent information.
-----------------------------------
FF                                  # EOF opcode.
```

- HTTP debug接口

由于opentracker响应内容是按BENCODE编码过的,调试时不太方便。因而扩展了一个返回human-readable内容的调试接口。

具体参看: https://github.com/flygoast/opentracker
