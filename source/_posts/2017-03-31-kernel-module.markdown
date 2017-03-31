---
layout: post
title: "Linux内核模块开发"
date: 2017-03-31 23:50:43 +0800
comments: true
categories: Kernel
---
Linux内核支持动态加载或卸载模块。这种机制使得不用重新编译内核就可以方便地扩展内核功能，也减小了内核镜像的大小。一般情况下，Linux设备驱动都以模块形式来实现，需要时再加载。

内核模块的扩展名为`.ko`, 一般位于`/lib/modules/<kernel_version>/kernel`目录下。

本文通过一个"hello world”级别的内核模块，来展示内核模块如何开发。

<!--more-->

首先，来看内核模块的操作命令。

* lsmod: 展示已经加载的内核模块

```plain
[root@centos1 vagrant]# lsmod
Module                  Size  Used by
nfnetlink_queue        18197  0
nfnetlink              14606  1 nfnetlink_queue
iptable_filter         12810  0
xt_NFQUEUE             12697  0
vboxsf                 39725  1
crct10dif_pclmul       14289  0
crc32_pclmul           13113  0
crc32c_intel           22079  0
ghash_clmulni_intel    13259  0
ppdev                  17671  0
…
```

* insmod: 加载内核模块

```plain
[root@centos1 hello]# insmod ./hello.ko
[root@centos1 hello]# lsmod |grep hello
hello                  12428  0
```

* modinfo: 展示模块文件信息

```plain
[root@centos1 hello]# modinfo hello.ko
filename:       /home/vagrant/hello/hello.ko
description:    A Simple Hello World module
author:         flygoast
license:        GPL
rhelversion:    7.1
srcversion:     2E67B30196A57711CD2CC3E
depends:
vermagic:       3.10.0-229.14.1.el7.x86_64 SMP mod_unload modversions
```

* rmmod: 移除内核模块, 需要注意的是，若该模块还在被其他模块使用，则不能被移除

```plain
[root@centos1 hello]# rmmod hello.ko
[root@centos1 hello]#
```

* modprobe: 基于模块依赖智能的添加或者移除内核模块，具体信息参考man modprobe

接下来，说明内核模块开发的流程。

编译内核模块时，需要内核开发包，我的环境为CentOS7，使用YUM命令安装:
```bash
yum install -y kernel-devel
```
创建一个目录`hello`, 并在其中编辑源代码文件`hello.c`, 内容如下:
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("flygoast");
MODULE_DESCRIPTION("A Simple Hello World module");

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello world!\n");
    return 0;
}

static void __exit hello_cleanup(void)
{
    printk(KERN_INFO "Cleaning up module.\n");
}

module_init(hello_init);
module_exit(hello_cleanup);
```
创建`Makefile`文件，内容如下:
```plain
obj-m += hello.o
all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
编译生成内核模块:
```plain
[root@centos1 hello]# make
make -C /lib/modules/3.10.0-229.14.1.el7.x86_64/build M=/home/vagrant/hello modules
make[1]: Entering directory `/usr/src/kernels/3.10.0-229.14.1.el7.x86_64'
  CC [M]  /home/vagrant/hello/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/vagrant/hello/hello.mod.o
  LD [M]  /home/vagrant/hello/hello.ko
make[1]: Leaving directory `/usr/src/kernels/3.10.0-229.14.1.el7.x86_64’
```
编译已经完成，我们动态加载和移除该模块:
```plain
[root@centos1 hello]# insmod hello.ko
[root@centos1 hello]# dmesg | tail -1
[62576.091311] Hello world!
[root@centos1 hello]# rmmod hello.ko
[root@centos1 hello]# dmesg | tail -1
[62611.981529] Cleaning up module.
```
内核模块被加载时，`module_init`宏会被调用。在我们模块中，它会调用`hello_init`函数在`/var/log/message`中输出字符串。与此类似，内核模块被移除时，`module_exit`宏会被调用，它会调用`hello_cleanup`函数完成模块资源的清理工作。

以上简要说明了内核模块开发的流程。在编写内核模块时，需要非常小心，一个小错误就会导致内核崩溃。若模块被移除时资源没有被完全释放，也会造成资源的浪费。

