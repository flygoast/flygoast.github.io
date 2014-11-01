---
layout: post
title: "PowerDNS模块机制分析"
date: 2014-11-01 11:15:45 +0800
comments: true
categories: DNS
---

[PowerDNS](http://www.powerdns.com/)是一个开源DNS服务器。它包括了授权DNS服务器和递归DNS服务器。本文只讨论授权DNS服务器，主要分析PowerDNS的架构及Backend模块机制的实现。

PowerDNS由各种各样的backend来处理DNS解析相关数据，数据可以存储在BIND模式的ZONE文件中，也可以存储在关系型数据库中。官方实现了很多backend，如Bind, Postgresql, Mysql等。每种backend实现为一个SO文件。PowerDNS启动时根据配置中的launch语句加载指定的so文件。这种方式使得PowerDNS的可扩展性非常高。
简略的PowerDNS架构图如下:

```
  +-----------------------------------------------------------------------+
  |                    |                 |                                |
  |    +------------+  |  +-----------+  |                                |
  |    |main thread |  |  |PacketCache|  |                                |
  |    +------------+  |  +-----------+  |                                |
  |                    |                 |                                |
  |    +------------+  |                 |                                |
  |    |DynListerner|  |                 |                                |
  |    +------------+  |                 |  +-----------+   +----------+  |
  |                    |       +---------+--|Distributor|---|DNSBackend|  |
  |    +--------+      |       |         |  +-----------+   +----------+  |
  |    |receiver|------+-------+         |                                |
  |    +--------+      |       |         |  +-----------+   +----------+  |
  |                    |       +---------+--|Distributor|---|DNSBackend|  |
  |       ...          |                 |  +-----------+   +----------+  |
  |                    |                 |     ...                        |
  |                    |                 |                                |
  |    +--------+      |                 |  +-----------+   +----------+  |
  |    |receiver|------+-------+---------+--|Distributor|---|DNSBackend|  |
  |    +--------+      |       |         |  +-----------+   +----------+  |
  |                    |       |         |                                |
  |                    |       |         |  +-----------+   +----------+  |
  |                    |       +---------+--|Distributor|---|DNSBackend|  |
  |                    |                 |  +-----------+   +----------+  |
  |                                                                       |
  +-----------------------------------------------------------------------+
```

下面具体分析PowerDNS的启动流程。

```c
    if(::arg().mustDo("guardian") && !isGuarded(argv)) {
      if(::arg().mustDo("daemon")) {
        L.toConsole(Logger::Critical);
        daemonize();
      }
      guardian(argc, argv);
      // never get here, guardian will reinvoke process
      cerr<<"Um, we did get here!"<<endl;
    }

    // we really need to do work - either standalone or as an instance
    ...
```

如果启动参数中指定了“--guardian=yes”，则PowerDNS运行在guardian模式，主进程fork出一个子进程执行正常的启动流程，而父进程则一秒钟探测一次子进程是否存活，如果子进程异常退出则重新启动子进程。

```c
      for(;;) {
        int ret=waitpid(pid,&status,WNOHANG);

        if(ret<0) {
          L<<Logger::Error<<"In guardian loop, waitpid returned error: "<<strerror(errno)<<endl;
          L<<Logger::Error<<"Dying"<<endl;
          exit(1);
        }
        else if(ret) // something exited
          break;
        else { // child is alive
          // execute some kind of ping here
          if(DLQuitPlease())
            takedown(1); // needs a parameter..
          setStatus("Child running on pid "+itoa(pid));
          sleep(1);
        }
      }

      pthread_mutex_lock(&g_guardian_lock);
      close(g_fd1[1]);
      fclose(g_fp);
      g_fp=0;

      if(WIFEXITED(status)) {
        int ret=WEXITSTATUS(status);

        if(ret==99) {
          L<<Logger::Error<<"Child requested a stop, exiting"<<endl;
          exit(1);
        }
        setStatus("Child died with code "+itoa(ret));
        L<<Logger::Error<<"Our pdns instance exited with code "<<ret<<endl;
        L<<Logger::Error<<"Respawning"<<endl;

        goto respawn;
      }
```

子进程或非guardian模式的原生进程执行正常的启动流程。

```c
    loadModules();
    BackendMakers().launch(::arg()["launch"]); // vrooooom!
```

loadModules()会根据"load-modules"配置语句调用dlopen来加载相应的SO文件。BackendMakers().launch(::arg()["launch"]);来加载launch语句指定的modules。然后开始监听指定的UDP和TCP端口。

```c
    N=new UDPNameserver; // this fails when we are not root, throws exception

    if(!::arg().mustDo("disable-tcp"))
      TN=new TCPNameserver;
```

接着调用mainthread()来初始化若干网络包接收线程。

```c
  unsigned int max_rthreads= ::arg().asNum("receiver-threads");
  for(unsigned int n=0; n < max_rthreads; ++n)
    pthread_create(&qtid,0,qthread, reinterpret_cast<void *>(n)); // receives packets
```

而每个接收线程则会创建若干自己的distributor线程。

```c
DNSDistributor *distributor = DNSDistributor::Create(::arg().asNum("distributor-threads")); // the big dispatcher!
```

接下来接收线程开始自己处理网络数据。接收到DNS查询包后，首先在PacketCache中查询是否有相应的响应。PacketCache是PowerDNS中Backend响应的Cache，它减少对于Backend的访问，可以大大提高性能。如果在PacketCache中没有找到，则调用distributor->question分发给distributor线程。示意代码如下:

```c
  for(;;) {
    if(!(P=NS->receive(&question))) { // receive a packet         inline
      continue;                    // packet was broken, try again
    }

    if(P->couldBeCached() && PC.get(P, &cached)) { // short circuit - does the PacketCache recognize this question?
      NS->send(&cached);   // answer it then
      continue;
    }

    distributor->question(P, &sendout);
  }
```

在调用question方法时会传递一个回调函数sendout。distributor线程处理完DNS请求后将调用该回调函数将响应返回给客户端。

```c
void sendout(const DNSDistributor::AnswerData &AD)
{
  static unsigned int &numanswered=*S.getPointer("udp-answers");
  static unsigned int &numanswered4=*S.getPointer("udp4-answers");
  static unsigned int &numanswered6=*S.getPointer("udp6-answers");

  if(!AD.A)
    return;

  N->send(AD.A);
  numanswered++;

  if(AD.A->d_remote.getSocklen()==sizeof(sockaddr_in))
    numanswered4++;
  else
    numanswered6++;

  int diff=AD.A->d_dt.udiff();
  avg_latency=(int)(1023*avg_latency/1024+diff/1024);

  delete AD.A;
}
```

distributor线程中调用backend来处理DNS响应。示意代码如下：

```c
    B.lookup;
    DNSResourceRecord rr;
    while(yb.get(rr))
        cout<<"Found cname pointing to '"+rr.content+"'"<<endl;
    }
```

其中B为UeberBackend实例。UeberBackend是一个特殊的Backend。启动时被加载的其他backend向它注册，被保存在一个vector中。UeberBackend按注册顺序依次调用其他backend的相应成员方法。以reload操作为例:

```c
void UeberBackend::reload()
{
  for ( vector< DNSBackend * >::iterator i = backends.begin(); i != backends.end(); ++i )
  {
    ( *i )->reload();
  }
}
```

lookup和get成员方法实现比较特殊。UeberBackend有一个成员变量d_handle表示当前处理请求的真实backend.在lookup中，会将第一个注册的backend赋给d_handle, 并调用该backend的lookup。

```c
(d_handle.d_hinterBackend=backends[d_handle.i++])->lookup(qtype, qname,pkt_p,zoneId);
```

UeberBackend的get方法会调用d_handle实例的get方法。而d_handle.get首先调用第一个backend的get方法。如果成功，则返回。否则调用下一个backend的lookup和get方法。直到get方法返回成功或者遍历完所有backend.这种实现保证了get方法所获取的响应是由本backend的lookup所生成。

```c
bool UeberBackend::handle::get(DNSResourceRecord &r)
{
  DLOG(L << "Ueber get() was called for a "<<qtype.getName()<<" record" << endl);
  bool isMore=false;
  while(d_hinterBackend && !(isMore=d_hinterBackend->get(r))) { // this backend out of answers
    if(i<parent->backends.size()) {
      DLOG(L<<"Backend #"<<i<<" of "<<parent->backends.size()
           <<" out of answers, taking next"<<endl);

      d_hinterBackend=parent->backends[i++];
      d_hinterBackend->lookup(qtype,qname,pkt_p,parent->domain_id);
    }
    else
      break;

    DLOG(L<<"Now asking backend #"<<i<<endl);
  }

  if(!isMore && i==parent->backends.size()) {
    DLOG(L<<"UeberBackend reached end of backends"<<endl);
    return false;
  }

  DLOG(L<<"Found an answering backend - will not try another one"<<endl);
  i=parent->backends.size(); // don't go on to the next backend
  return true;
}
```

下面介绍module加载的过程。PowerDNS的module实现需要包含三个部分。

1) Backend本身，它需要继承父类DNSBackend.

```c
/* FIRST PART */
class RandomBackend : public DNSBackend{}
```
2) 工厂类，它用于产生Backend实例。

```c
/* SECOND PART */
class RandomFactory : public BackendFactory{}
```
3) 加载器类和该类的一个static实例。

```c
/* THIRD PART */
class RandomLoader{};
static RandomLoader randomloader;
```

当进程调用dlopen时，该static loader实例的构造函数被调用，将其注册在UeberBackend中。

可以参考我写的一个简单的PowerDNS模块: [remoteipbackend](https://github.com/flygoast/remoteipbackend), 用于返回到达它的递归DNS的出口IP地址。
