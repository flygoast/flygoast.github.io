---
layout: post
title: "XenServer编译内核模块"
date: 2016-11-06 17:25:14 +0800
comments: true
categories: xen
---
Citrix提供了DDK(Driver Development Kit)来支持在XenServer中要构建自定义的内核模块或硬件驱动。DDK是一个OVA格式的虚拟机镜像，包含了内核头文件和编译器等开发工具。

下面介绍使用DDK构建内核模块的步骤。

首先从官方下载相应版本DDK，这里选择6.5:
http://downloadns.citrix.com.edgesuite.net/10106/XenServer-6.5.0-DDK.iso

将下载的ISO文件上传到XenServer宿主机上

挂载ISO
```bash
mkdir /mnt/tmp
mount <path_to_DDK>/ddk.iso /mnt/tmp -o loop
```

<!--more-->

使用DDK镜像创建虚拟机
```bash
xe vm-import filename=/mnt/tmp/ova.xml
```
`xe vm-import`命令会用该镜像创建一个虚拟机，并会输出该虚拟机的UUID，如:
```bash
[root@xenserver-iryatlxz ~]# xe vm-import filename=/mnt/tmp/ddk/ova.xml
69a2356e-5f7f-0fd8-a609-234a28b59fc5
```

接下来找到eth0关联的网络UUID。
首先列出所有网络:
```bash
xe network-list
```
输出如下:
```plain
uuid ( RO)                : e0f9ba3d-f27b-7380-413a-0491db9e0ec4
          name-label ( RW): Pool-wide network associated with eth1
    name-description ( RW):
              bridge ( RO): xenbr1


uuid ( RO)                : 46fb28dd-4c35-5755-160b-f6389e09c54a
          name-label ( RW): Pool-wide network associated with eth0
    name-description ( RW):
              bridge ( RO): xenbr0


uuid ( RO)                : 0ecf8369-5469-1327-2195-f3cc28a1b3bd
          name-label ( RW): Host internal management network
    name-description ( RW): Network on which guests will be assigned a private link-local IP address which can be used to talk XenAPI
              bridge ( RO): xenapi
```
从中找到eth0关联的网络UUID，为:
```plain
46fb28dd-4c35-5755-160b-f6389e09c54a
```

使用上面得到的网络UUID和虚拟机UUID创建虚拟接口:
```bash
xe vif-create network-uuid=46fb28dd-4c35-5755-160b-f6389e09c54a  vm-uuid=69a2356e-5f7f-0fd8-a609-234a28b59fc5 device=0
```

启动虚拟机
```bash
xe vm-start uuid=69a2356e-5f7f-0fd8-a609-234a28b59fc5
```

可以使用XenCenter的控制台来访问DDK虚拟机，也可以直接在命令行使用xenconsole来访问。

使用xenconsole访问的步骤如下:

获取domain ID
```bash
[root@xenserver-iryatlxz ~]# xe vm-list params=dom-id uuid=69a2356e-5f7f-0fd8-a609-234a28b59fc5 --minimal
14
```

从console连接该虚拟机
/usr/lib64/xen/bin/xenconsole 14

登录到DDK VM后，就可以在该虚拟机中构建自定义的内核模块或硬件驱动了。内核开发包位于`/usr/src/kernels/3.10.0+2-x86_64/`。


需要退出时，按`CTRL-]`退出。
