---
title: C/S架构下UUID冲突处理方案简要说明
date: 2022-08-11 23:10:10
categories: MISC
---
在`C/S`架构中，`Server`端需要依赖`ID`来唯一标识`Client`，而客户端相关数据都由这个唯一标识来索引。

这个标识可以由服务端分配，也可以由客户端自行生成再注册到服务端。

服务端分配是一种中心化生成方式，可以保证唯一性。客户端自行生成方式可以采用`UUID`标准来生成，也可以保证唯一性。

尽管在生成方式可以保证唯一性，但是由于客户端某些特性的改变，服务端依然会出现`ID`冲突的场景。

<!--more-->

比如，客户端在虚拟机中安装完成并已生成好`UUID`, 此时克隆虚拟机，克隆机中的客户端启动并使用原`UUID`与服务端通信。此时，如果服务端不使用客户端的属性进行校验，那么两个客户端在服务端就会被识别为一个客户端，从而可能造成数据混乱和覆盖。

于是，服务端需要对客户端的某些属性进行校验来检测`UUID`冲突，常用的属性如`MAC`地址，`IP`地址等信息。如果已经正常运行的客户端所在的机器属性发生变化时，比如，更换网卡，`MAC`地址发生变化，更改`IP`地址，由于服务端会校验这些属性，在服务端看来这些变化的属性也造成`UUID`冲突。

尽管都是相同的`UUID`，第一种情形，需要视为不同的客户端进行处理，第二种情况则需要视为同一个客户端进行处理。而单一从服务端角度是无法区分这两种情形的，需要引入人为判断，或者服务端根据业务场景提前配置好`UUID`冲突的处置方案。

针对这种冲突处置方案，简单梳理了一下客户端和服务端的处理流程:

客户端流程图:
{% img /images/2022-08-11/1.png %}

控制中心流程图:
{% img /images/2022-08-11/2.png %}

在不同场景下，可以根据实际业务需要选择需要校验的属性。

而在`Linux`上具体生成`UUID`的方法有许多，常用的方法有:

* 通过`dmidecode`工具获取:

```bash
[root@default ~]# dmidecode -s system-uuid
cfba65ba-62dd-42f1-8976-8f3653de090a
```

* 从`sysfs`中获取:

```bash
[root@default ~]# cat /sys/class/dmi/id/product_uuid
CFBA65BA-62DD-42F1-8976-8F3653DE090A
```

* 随机生成:

```bash
[root@default ~]# cat /proc/sys/kernel/random/uuid
396fea55-23bb-4407-adf5-b527a117e8e8
```

参考:
* https://www.uwintech.cn/blog/easyops-cmdb
* https://liuzehe.top/archives/2021083101
* https://www.tremplin-numerique.org/zh-CN/que-sont-les-uuid-et-pourquoi-sont-ils-utiles-informatique-cloudsavvy
* https://en.wikipedia.org/wiki/Universally_unique_identifier
