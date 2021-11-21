---
title: QEMU虚拟机内识别ivshmem设备
date: 2021-09-12 12:56:34
tags:
- Virtualization
- Ivshmem
- Qemu
categories: Virtualization
description:
---

`ivshmem:(Inter-VM shared memory device)`是`QEMU`提供的一种宿主机与虚拟机之间或多个虚拟机之间共享内存的特殊设备。它有两种形式:
* `ivshmem-plain`: 简单的共享内存区域
* `ivshmem-doorbell`: 除了共享内存，还能提供基于中断的通信机制

这种设备在虚拟机内部表现为`PCI`设备，共享的内存区域则以`PCI BAR`的形式存在。`ivshmem`PCI设备提供3个`BAR`:
* `BAR0`: 设备寄存器
* `BAR1`: `MSI-X`表
* `BAR2`: 共享内存区域

简单共享内存的场景只使用`BAR2`就足够了。如果需要基于中断实现额外通信，需要用到`BAR0`和`BAR1`。这可能需要编写内核驱动在虚拟机内处理中断，宿主机上`QEMU`进程在启动前需要先启动`ivshmem server`, 然后让`QEMU`进程连接到`server`的`unix socket`。

具体可以参考[官方文档](:https://github.com/qemu/qemu/blob/master/docs/specs/ivshmem-spec.txt)。

本文只讨论`ivshmem-plain`模式。宿主机上添加`ivshmem`设备后，虚拟机应用如何找到相应的`ivshmem`设备呢？

Linux的`/sys/bus/pci/devices/`目录会列出所有的PCI设备，`ivshmem`设备也会包含在其中。PCI设备都存在`vendor`号和`device`两个标识，`vendor`表示厂商，`device`表示厂商内的设备类型。`ivshmem`设备的`vendor`号为`0x1af4`, `device`号为`0x1110`，PCI设备的`vendor`和`device`号可在[这里](https://pci-ids.ucw.cz/read/PC?restrict=)进行查询。

虚拟机中应用可通过遍历该目录下的具体设备，通过读取`vendor`和`device`文件来识别`ivshmem`设备。

但如果有两种应用都需要使用一个独立的`ivshmem`设备，虚拟机应用如何识别出应该使用哪个`ivshmem`设备呢?

因为每个PCI设备都可以由`BDF:(Bus, Device, Function)`来唯一标识，简单做法可以为每个应用预留好固定`BDF`地址。`BDF`地址中，`BUS`占用`8`位，`Device`占用`5`位,`Function`占用`3`位。比如，预留总线`pci0`的最后两个设备地址`0000:00:1e.0`和`0000:00:1f.0`。

有时候无法预留，不同虚拟机上的`ivshmem`地址可能不同。这种情况可以通过与宿主机上的应用约定好相应的固定内容做为`signature`写入共享内存头部，虚拟机应用读取共享内存头部的`signature`信息来识别相应设备。

<!--more-->

之前的文章[<<QEMU monitor机制实例介绍>>](/2017/11/19/qemu-monitor/)介绍了`QEMU`的监控机制。我们使用可以该机制动态添加`ivshmem`设备。

首先，识别虚拟机当前的PCI设备:
```plain
[root@controller ~]# virsh qemu-monitor-command --hmp 4 info pci
  Bus  0, device   0, function 0:
    Host bridge: PCI device 8086:1237
      id ""
  Bus  0, device   1, function 0:
    ISA bridge: PCI device 8086:7000
      id ""
  Bus  0, device   1, function 1:
    IDE controller: PCI device 8086:7010
      BAR4: I/O at 0xc0a0 [0xc0af].
      id ""
  Bus  0, device   1, function 2:
    USB controller: PCI device 8086:7020
      IRQ 11.
      BAR4: I/O at 0xc040 [0xc05f].
      id "usb"
  Bus  0, device   1, function 3:
    Bridge: PCI device 8086:7113
      IRQ 9.
      id ""
  Bus  0, device   2, function 0:
    VGA controller: PCI device 1013:00b8
      BAR0: 32 bit prefetchable memory at 0xfc000000 [0xfdffffff].
      BAR1: 32 bit memory at 0xfebd0000 [0xfebd0fff].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0000fffe].
      id "video0"
  Bus  0, device   3, function 0:
    Ethernet controller: PCI device 1af4:1000
      IRQ 11.
      BAR0: I/O at 0xc060 [0xc07f].
      BAR1: 32 bit memory at 0xfebd1000 [0xfebd1fff].
      BAR4: 64 bit prefetchable memory at 0xfe000000 [0xfe003fff].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0003fffe].
      id "net0"
  Bus  0, device   4, function 0:
    SCSI controller: PCI device 1af4:1001
      IRQ 11.
      BAR0: I/O at 0xc000 [0xc03f].
      BAR1: 32 bit memory at 0xfebd2000 [0xfebd2fff].
      BAR4: 64 bit prefetchable memory at 0xfe004000 [0xfe007fff].
      id "virtio-disk0"
  Bus  0, device   5, function 0:
    Class 0255: PCI device 1af4:1002
      IRQ 10.
      BAR0: I/O at 0xc080 [0xc09f].
      BAR4: 64 bit prefetchable memory at 0xfe008000 [0xfe00bfff].
      id "balloon0"
```


PCI地址使用到了`0000:00:05.0`, 我们使用未使用的PCI地址添加两个`ivshmem`设备，`0000:00:10.0`大小为`16M`, `0000:00:11.0`大小为`8M`:
```bash
virsh qemu-monitor-command --hmp 4 "object_add  memory-backend-file,size=16M,share,mem-path=/dev/shm/shm1,id=shm1"
virsh qemu-monitor-command --hmp 4 "device_add  ivshmem-plain,memdev=shm1,bus=pci.0,addr=0x10,master=on"
virsh qemu-monitor-command --hmp 4 "object_add  memory-backend-file,size=8M,share,mem-path=/dev/shm/shm2,id=shm2"
virsh qemu-monitor-command --hmp 4 "device_add  ivshmem-plain,memdev=shm2,bus=pci.0,addr=0x11,master=on"
```

登录到虚拟机上查看PCI设备，可以看到:
```plain
[root@host-172-16-0-29 ~]# lspci
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma] (rev 02)
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:01.1 IDE interface: Intel Corporation 82371SB PIIX3 IDE [Natoma/Triton II]
00:01.2 USB controller: Intel Corporation 82371SB PIIX3 USB [Natoma/Triton II] (rev 01)
00:01.3 Bridge: Intel Corporation 82371AB/EB/MB PIIX4 ACPI (rev 03)
00:02.0 VGA compatible controller: Cirrus Logic GD 5446
00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device
00:04.0 SCSI storage controller: Red Hat, Inc. Virtio block device
00:05.0 Unclassified device [00ff]: Red Hat, Inc. Virtio memory balloon
00:10.0 RAM memory: Red Hat, Inc. Inter-VM shared memory (rev 01)
00:11.0 RAM memory: Red Hat, Inc. Inter-VM shared memory (rev 01)
```

查看目录`/sys/bus/pci/devices/`, 也可以看这些设备:

```
[root@host-172-16-0-29 ~]# ls -l /sys/bus/pci/devices/
total 0
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:00.0 -> ../../../devices/pci0000:00/0000:00:00.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:01.0 -> ../../../devices/pci0000:00/0000:00:01.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:01.1 -> ../../../devices/pci0000:00/0000:00:01.1
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:01.2 -> ../../../devices/pci0000:00/0000:00:01.2
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:01.3 -> ../../../devices/pci0000:00/0000:00:01.3
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:02.0 -> ../../../devices/pci0000:00/0000:00:02.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:03.0 -> ../../../devices/pci0000:00/0000:00:03.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:04.0 -> ../../../devices/pci0000:00/0000:00:04.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:39 0000:00:05.0 -> ../../../devices/pci0000:00/0000:00:05.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:47 0000:00:10.0 -> ../../../devices/pci0000:00/0000:00:10.0
lrwxrwxrwx. 1 root root 0 Sep 12 08:47 0000:00:11.0 -> ../../../devices/pci0000:00/0000:00:11.0
```

分别查看两个`ivshmem`设备目录下`vendor`和`device`文件的内容，可以看到`vendor`都是`0x1af4`,`device`都是`0x1110`:
```plain
[root@host-172-16-0-29 ~]# cat /sys/bus/pci/devices/0000\:00\:10.0/vendor
0x1af4
[root@host-172-16-0-29 ~]# cat /sys/bus/pci/devices/0000\:00\:10.0/device
0x1110
[root@host-172-16-0-29 ~]# cat /sys/bus/pci/devices/0000\:00\:11.0/vendor
0x1af4
[root@host-172-16-0-29 ~]# cat /sys/bus/pci/devices/0000\:00\:11.0/device
0x1110
```

再查看一个其他PCI设备`0000:00:05.0`的`device`值为`0x1002`, 不是`0x1110`:
```
[root@host-172-16-0-29 ~]# cat /sys/bus/pci/devices/0000\:00\:05.0/device
0x1002
```

`ivshmem`设备的共享内存以设备目录下的`resource2`文件存在，虚拟机应用可以通过`mmap`调用来读写该内存区域。查看两个`ivshmem`共享内存的大小,可以看到`0000:00:10.0`的大小为`16M`, `0000:00:11.0`的大小为`8M`:
```plain
[root@host-172-16-0-29 ~]# ls -l /sys/bus/pci/devices/0000\:00\:10.0/resource2
-rw-------. 1 root root 16777216 9月  12 09:23 /sys/bus/pci/devices/0000:00:10.0/resource2
[root@host-172-16-0-29 ~]# ls -l /sys/bus/pci/devices/0000\:00\:11.0/resource2
-rw-------. 1 root root 8388608 9月  12 11:11 /sys/bus/pci/devices/0000:00:11.0/resource2
```

在宿主机上分别在两个共享内存区域中写入`8`字节(含字符串换行符)的不同标识信息:
```bash
echo "SIGN_01" > /dev/shm/shm1
echo "SIGN_02" > /dev/shm/shm2
```

在虚拟机上编写一个简单的程序来读出第一个`ivshmem`设备的前`8`字节, `C`语言代码如下:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <assert.h>


#define SHM_SIZE  (16 * 1024 * 1024)


int main(int argc, char **argv) {
    char *p;
    int fd;
    int i;

    fd = open("/sys/bus/pci/devices/0000:00:10.0/resource2", O_RDWR);
    assert(fd != -1);

    p = mmap(0, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    assert(p != NULL);

    for (i = 0; i < 8; i++) {
        printf("%c", p[i]);
    }

    munmap(p, SHM_SIZE);
    close(fd);

    return 0;
}
```

在虚拟机上编译执行, 可以看到宿主机上写入的标识信息。
```plain
[root@host-172-16-0-29 ~]# gcc t.c
[root@host-172-16-0-29 ~]# ./a.out
SIGN_01
```

在真实的生产应用中，对于共享内存的使用不会这么简单，而是要构造相当复杂的数据结构。比如，可以在共享内存中构造基于偏移量的环形队列结构，用于双向的信息发送。

基于中断方式的通信方式，后续再专门文章来介绍。

参考文档:

* https://www.linux-kvm.org/images/e/e8/0.11.Nahanni-CamMacdonell.pdf
* https://www.dazhuanlan.com/mcalf/topics/1478919
* https://www.kernel.org/doc/Documentation/filesystems/sysfs-pci.txt
* https://github.com/virtio-win/kvm-guest-drivers-windows/tree/master/ivshmem
* https://cloud.tencent.com/developer/article/1593168
