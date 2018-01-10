---
layout: post
title: "虚拟机KPTI性能影响分析"
date: 2018-01-11 03:52:34 +0800
comments: true
categories: Virtulization
---
`Meltdown`漏洞暴发之后，各云厂商已陆陆续续升级宿主机，升级`Guest`是否有较大的性能影响, 是否应该升级`Guest`虚拟机等问题又困扰着云租户们。

本文基于实例来分析`Guest`虚拟机的性能影响。

首先来看`Meltdown`的原理。现代CPU架构在不同的安全等级运行指令。内核需要直接操作硬件，运行于最高权限，而几乎所有的应用程序都运行于各低权限上。用户态进程通过系统调用访问内核数据。一般情况下，CPU在执行时使用虚拟内存地址，内核使用页表来控制虚拟内存与物理内存的映射。用户态进程不能访问映射给内核的内存页。这些映射关系的计算较为耗时，因而CPU使用`TLB(Translation lookaside buffer)`来存储计算后的映射关系。从`TLB`中移除这些映射再重新计算并缓存会消耗性能，因而内核实现上会尽可能减少刷新TLB，于是内核将用户态和内核的数据都映射进用户态页表中。正常情况下，这不存在安全风险，页表的权限可以阻止用户态进程访问内核数据。

<!--more-->

现代CPU追求运行速度，会从所有的内存映射关系中预取数据，而CPU本身会乱序执行，且乱序执行时不会验证是否有权限访问内存，且CPU缓存不会被恢复，因而`Meltdown`的研究者构造乱序执行的场景将内核内存地址的内容获取到CPU缓存中，再通过侧信道攻击获取到相应内容。

Linux内核采用`KPTI(Kernel Page Table Isolation)`机制，将内核页表和用户空间页表隔离开，避免CPU在预取和乱序执行时用户态进程时获取到内核页表中的数据，从而缓解`Meltdown`利用。

这样当进程用户在用户空间时，使用用户态页表，发生中断或者异常陷入内核时，将切换到内核页表。页表的切换将导致性能下降。

Intel在`Haswell`架构后增加了一个特性叫做`PCID(Process-Context Identifiers)`。在没有该特性时，切换页表时将需要刷新整个`TLB`，而在CPU支持PCID时，内核用不同ID来区分两个页表将其关联PCID不再需要刷新整个TLB，这样会大大减小页表切换带来的性能损失。

在Linux中查看`PCID`的方法:
```bash
cat /proc/cpuinfo |grep pcid
```
根据上述分析，如果`Guest`虚拟机的CPU支持`PCID`，则`KPTI`引发的性能下降应会更小一些。。`VMware ESXi`平台上的`Guest`虚拟机CPU支持`PCID`，而KVM平台上`Guest`虚拟机CPU是否支持`PCID`由`QEMU/KVM`进程的`-cpu`选项决定。

查看KVM支持的CPU选项:
```bash
root@dummy:~# /usr/bin/kvm -cpu help
x86           qemu64  Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz
x86           phenom  AMD Phenom(tm) 9550 Quad-Core Processor
x86         core2duo  Intel(R) Core(TM)2 Duo CPU     T7700  @ 2.40GHz
x86            kvm64  Common KVM processor
x86           qemu32  Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz
x86            kvm32  Common 32-bit KVM processor
x86          coreduo  Genuine Intel(R) CPU           T2600  @ 2.16GHz
x86              486
x86          pentium
x86         pentium2
x86         pentium3
x86           athlon  Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz
x86             n270  Intel(R) Atom(TM) CPU N270   @ 1.60GHz
x86           Conroe  Intel Celeron_4x0 (Conroe/Merom Class Core 2)
x86           Penryn  Intel Core 2 Duo P9xxx (Penryn Class Core 2)
x86          Nehalem  Intel Core i7 9xx (Nehalem Class Core i7)
x86         Westmere  Westmere E56xx/L56xx/X56xx (Nehalem-C)
x86      SandyBridge  Intel Xeon E312xx (Sandy Bridge)
x86          Haswell  Intel Core Processor (Haswell)
x86        Broadwell  Intel Core Processor (Broadwell)
x86       Opteron_G1  AMD Opteron 240 (Gen 1 Class Opteron)
x86       Opteron_G2  AMD Opteron 22xx (Gen 2 Class Opteron)
x86       Opteron_G3  AMD Opteron 23xx (Gen 3 Class Opteron)
x86       Opteron_G4  AMD Opteron 62xx class CPU
x86       Opteron_G5  AMD Opteron 63xx class CPU
x86             host  KVM processor with all supported host features (only available in KVM mode)

...
```

若需要`Guest`支持`PCID`, 启动`QEMU/KVM`进程时，需要将`-cpu`选项设置为`Haswell`, `Broadwell`或者使用`host-model`,`host-passthrough`等模式。


在我们的KVM平台创建了一台`CentOS6.8`的`Guest`虚拟机来验证一下升级更新支持KPTI后在是否支持PCID两种情况下的性能影响。

测试方式为记录在该虚拟机上执行编译内核的时间，测试两种情况:

* 原生内核
* 支持`KPTI`的内核，升级完后内核版本为: `2.6.32-696.18.7.el6.x86_64`

在虚拟机CPU不支持`PCID`的情况下测试结果如下:

1）原生内核:
```plain
       real       10m10.618s
       user       9m10.398s
       sys        0m55.487s
```

2) `KPTI`内核:
```plain
       real       11m10.861s
       user       9m51.371s
       sys        1m22.248s
```
升级后性能下降约9.8%。

在虚拟机CPU支持`PCID`的情况下测试结果如下:

1）原生内核:
```plain
       real    10m24.469s
       user    9m12.507s
       sys     0m59.281s
```

2) `KPTI`内核:
```plain
       real    10m40.750s
       user    9m19.495s
       sys     1m11.286s
```

升级后性能下降约2.5%


从我们的实例可以看到，`Guest`虚拟机CPU是否支持`PCID`对于性能下降有较大影响。

若用户的`Guest`虚拟机CPU不支持`PCID`可与云提供方沟通，并根据实际业务负载进行性能影响评估，综合考虑安全与性能决定是否升级更新Guest虚拟机。

在升级完Guest性能测试不理想的情况下，也可以在内核启动参数加添加`noibrs noibpb nopti`等参数禁用该特性。
