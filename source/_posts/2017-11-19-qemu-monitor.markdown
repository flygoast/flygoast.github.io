---
layout: post
title: "QEMU monitor机制实例分析"
date: 2017-11-19 14:40:08 +0800
comments: true
categories: Virtualization
---
QEMU实例运行时，用户可以通过monitor机制来与实例进行交互，通过它可以获取当前运行的虚拟机信息，处理热插拔设备，管理虚拟机快照等。要了解全部能力，可以参考文档:
https://qemu.weilnetz.de/doc/qemu-doc.html#pcsys_005fmonitor

QEMU启动时，需要使用`-monitor`选项指定做为`console`设备，官方文档说明如下:
```plain
-monitor dev
    Redirect the monitor to host device dev (same devices as the serial port). The default device is vc in graphical mode and stdio in non graphical mode. Use -monitor none to disable the default monitor.
```
下面首先以标准输入输出设备做为`console`来启动QEMU实例:
```plain
[root@localhost ~]# qemu-system-x86_64 cirros-0.3.5-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :20 -device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 -monitor stdio

QEMU 2.0.0 monitor - type 'help' for more information
(qemu)
```
在`console`里可以输入相关命令来完成我们的操作，比如我们查看虚拟机网络设备状态:
```plain
(qemu) info network
virtio-net-pci.0: index=0,type=nic,model=virtio-net-pci,macaddr=52:54:00:12:34:56
 \ net0: index=0,type=tap,ifname=tap0,script=no,downscript=no
```

<!--more-->

也可以动态添加设备，比如我们添加一个8M大小的`ivshmem`设备:
```plain
(qemu) device_add ivshmem,size=8m,shm=flygoast_vm0,bus=pci.0,addr=0x1f
(qemu) info pci
  Bus  0, device   0, function 0:
    Host bridge: PCI device 8086:1237
      id ""
  Bus  0, device   1, function 0:
    ISA bridge: PCI device 8086:7000
      id ""
  Bus  0, device   1, function 1:
    IDE controller: PCI device 8086:7010
      BAR4: I/O at 0xc020 [0xc02f].
      id ""
  Bus  0, device   1, function 3:
    Bridge: PCI device 8086:7113
      IRQ 9.
      id ""
  Bus  0, device   2, function 0:
    VGA controller: PCI device 1013:00b8
      BAR0: 32 bit prefetchable memory at 0xfc000000 [0xfdffffff].
      BAR1: 32 bit memory at 0xfebd0000 [0xfebd0fff].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0000fffe].
      id ""
  Bus  0, device   3, function 0:
    Ethernet controller: PCI device 1af4:1000
      IRQ 11.
      BAR0: I/O at 0xc000 [0xc01f].
      BAR1: 32 bit memory at 0xfebd1000 [0xfebd1fff].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0003fffe].
      id ""
  Bus  0, device  31, function 0:
    RAM controller: PCI device 1af4:1110
      IRQ 0.
      BAR0: 32 bit memory at 0x80800000 [0x808000ff].
      BAR2: 64 bit prefetchable memory at 0x80000000 [0x807fffff].
      id “"
```
执行后，我们在Guest OS里查看PCI设备, 可以看到已经检测到了新的PCI设备:

{% img /images/2017-11-19/1.png %}

除了标准输入输出设备，也可以使用网络连接做为`console`, 比如TCP、UnixSocket等。下面使用TCP监听端口做为`console`启动QEMU实例:
```plain
[root@localhost ~]# qemu-system-x86_64 cirros-0.3.5-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :20 -device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 -monitor tcp:127.0.0.1:4444,server,nowait -daemonize
```
使用`nc`连接`console`并查询虚拟机状态:
```plain
[root@localhost ~]# nc 127.0.0.1 4444
QEMU 2.0.0 monitor - type 'help' for more information
(qemu) info status
info status
VM status: running
(qemu)
```

上述这种方式更偏向用户直接输入命令进行交互，称为HMP(Human Machine Protocol)，程序使用这种方式不是太方便。QEMU还提供了另外一种基于JSON的QMP(QEMU Machine Protocol)来满足自动化处理的需求。Libvirt就是使用QMP来控制QEMU实例。

QMP规范可以参考:

* https://github.com/qemu/qemu/blob/master/docs/interop/qmp-intro.txt
* https://github.com/qemu/qemu/blob/master/docs/interop/qmp-spec.txt

QMP协议的工作流程如下:

* 连接建立后服务器发送欢迎信息，进入能力协商(`capabilities negotiation`)模式
* 客户端发送`{“execute”:”qmp_capablities”}`
* 成功则服务器返回`{“return”:{}}`，否则`return`中会含有`error`。
* 客户端发送命令
* 服务器以异步消息返回结果

QMP方式`console`也可以使用多种设备形式, 如，标准输入输出、TCP、UnixSocket等。可以通过QEMU选项`-mon`来指定`console`设备， 我们以标准输入输出设备做为`console`启动QEMU实例:
```plain
[root@localhost ~]# qemu-system-x86_64 cirros-0.3.5-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :20 -device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 -chardev stdio,id=mon0 -mon chardev=mon0,mode=control,pretty=on

{
    "QMP": {
        "version": {
            "qemu": {
                "micro": 0,
                "minor": 0,
                "major": 2
            },
            "package": ""
        },
        "capabilities": [
        ]
    }
}
```
可以看到服务器发送了欢迎信息到标准输出，我们在标准输入设备里输入:
```json
{"execute":"qmp_capabilities”}
```
QEMU实例返回:
```json
{
    "return": {
    }
}
```
此时我们可以发送命令了，我们来查询虚拟机状态:
```json
{"execute":"query-status”}
```
服务器返回了结果:
```json
{
    "return": {
        "status": "running",
        "singlestep": false,
        "running": true
    }
}
```

我们还可以使用UnixSocket做为`console`:
```plain
[root@localhost ~]# qemu-system-x86_64 cirros-0.3.5-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :20 -device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 -chardev socket,id=mon0,path=/tmp/vm0.monitor,server,nowait -mon chardev=mon0,mode=control,pretty=on -daemonize
```
使用`nc`连接UnixSocket文件, QEMU实例返回了欢迎信息:
```plain
[root@localhost ~]# nc -U /tmp/vm0.monitor
{
    "QMP": {
        "version": {
            "qemu": {
                "micro": 0,
                "minor": 0,
                "major": 2
            },
            "package": ""
        },
        "capabilities": [
        ]
    }
}
```
除了使用`-mon`选项，还可以直接使用`-qmp`选项:
```plain
[root@localhost ~]# qemu-system-x86_64 cirros-0.3.5-x86_64-disk.img -smp 2,cores=2 -m 2G -vnc :20 -device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -name vm0 -qmp unix:/tmp/vm0.monitor,server,nowait -daemonize
```
使用`nc`连接:
```plain
[root@localhost ~]# nc -U /tmp/vm0.monitor
{"QMP": {"version": {"qemu": {"micro": 0, "minor": 0, "major": 2}, "package": ""}, "capabilities": []}}
```
上面提到，Libvirt使用QMP与QEMU实例通信，libvirt创建QEMU实例时会指定一个UnixSocket文件做为`console`。我们可以通过使用`virsh`命令的`qemu-monitor-command`子命令来访问QEMU monitor，比如我们查询块文件信息:
```plain
[root@localhost ~]# virsh qemu-monitor-command i1 '{"execute":"query-block"}'
{"return":[{"io-status":"ok","device":"drive-ide0-0-0","locked":false,"removable":false,"inserted":{"iops_rd":0,"detect_zeroes":"off","image":{"virtual-size":21474836480,"filename":"/tmp/i1.qcow2","cluster-size":65536,"format":"qcow2","actual-size":12482580480,"format-specific":{"type":"qcow2","data":{"compat":"0.10","refcount-bits":16}},"dirty-flag":false},"iops_wr":0,"ro":false,"backing_file_depth":0,"drv":"qcow2","iops":0,"bps_wr":0,"write_threshold":0,"encrypted":false,"bps":0,"bps_rd":0,"cache":{"no-flush":false,"direct":false,"writeback":true},"file":"/tmp/i1.qcow2","encryption_key_missing":false},"type":"unknown"},{"io-status":"ok","device":"drive-ide0-0-1","locked":false,"removable":true,"tray_open":false,"type":"unknown"}],"id":"libvirt-13”}
```
这种方式使用的是JSON格式的QMP协议，可以加上`—hmp`选项直接输入命令来交互:
```plain
[root@localhost ~]# virsh qemu-monitor-command --hmp i1 info kvm
kvm support: enabled
```

我们如果想直接连接libvirt生成的Unix Socket文件来操作QEMU实例，需要先将libvirtd关闭，如:
```plain
[root@localhost ~]# service libvirtd stop
Redirecting to /bin/systemctl stop  libvirtd.service
[root@localhost ~]# nc -U /var/lib/libvirt/qemu/domain-26-i1/monitor.sock
{"QMP": {"version": {"qemu": {"micro": 0, "minor": 3, "major": 2}, "package": " (qemu-kvm-ev-2.3.0-31.0.el7_2.21.1)"}, "capabilities": []}}
^C
[root@localhost ~]# service libvirtd start
Redirecting to /bin/systemctl start  libvirtd.service
```



