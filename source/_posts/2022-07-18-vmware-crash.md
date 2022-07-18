---
title: VMware Guest虚拟机失去响应的排查方法
date: 2022-07-18 21:38:48
tags:
- Virtualization
- Crash
categories: Virtualization
---
在一些业务环境的VMware `Guest`虚拟机上安装我们的内核模块后，偶尔虚拟机会变成无响应的状态。重启后登录系统查看，处于无响应状态的那段时间在`/var/log/messages`中却没有任何信息，完全没有分析问题原因的头绪。

经过调研，找到两种方法可以帮助排查定位这类系统失去响应的问题，分别是:

* 发送`NMI: Non-Maskable Interupt`给虚拟机
* VMware内存快照

<!--more-->

#### 发送NMI给虚拟机

`NMI`是不可屏蔽的硬件中断。在虚拟化环境中，`Hypervisor`可以向`Guest`虚拟机发送`NMI`中断。`Guest`虚拟机收到`NMI`后产生`dump`文件, 继而从`dump`文件中分析系统状态。

只是`Linux`系统对于`NMI`中断的默认处理行为是发送到`stdout`，并不会`panic`并产生`dump`文件。为了在接收到`NMI`时产生`dump`文件，我们需要在`Guest`的`Linux`系统中完成这些配置:

* 安装`kexec-tools`, 并启动`kdump`服务:
```bash
systemctl start kdump.service
```

* 配置`sysctl`参数，令`Linux`系统接收到`NMI`时`panic`:
```bash
sysctl -w kernel.panic_on_unrecovered_nmi=1
sysctl -w kernel.unknown_nmi_panic=1
```

在`VCenter`的`GUI`界面中, 可以在相应`Guest`虚拟机的`导出系统日志`操作中向虚拟机发送`NMI`, 如图:
{% img /images/2022-07-18/1.png %}

这种方式对`VCenter`登录帐号的管理权限有所要求。

在相应的`ESXi`主机上也可以通过命令行向虚拟机发送`NMI`:
```bash
vm-support -a HungVM:Send_NMI_To_Guest --vm=/vmfs/volumes/Path/of/VMname.vmx
```

可以参考官方KB:
* https://kb.vmware.com/s/article/2149185

如果是`RHEV`和`oVirt`环境, 可以使用`virsh`命令发送`NMI`:
```bash
virsh inject-nmi guest1
```

具体可以参考链接:
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-generic_commands-injecting_nmi

#### 内存快照

另外一种方法就是基于VMware的快照。VMware官方提供了一个工具可以将快照文件转换成`crash`工具可以读取的`dump`文件。

工具使用说明:
* https://kb.vmware.com/s/article/2003941

工具下载地址:
* https://flings.vmware.com/vmss2core

但经过尝试，发现转换完成的`dump`文件使用`crash`并不能正常读取。

和`Redhat`这篇`KB`上描述的错误一致:
* https://access.redhat.com/solutions/3622951

`crash`无法读取的根本原因是从`7.5`版本开始，`RHEL`内核默认启用了`KASLR`特性:
```
Starting 7.5, RHEL kernels feature KASLR (Kernel Address Space Linear Randomization) enabled by default. KASLR is a security feature that enables the kernel to relocate itself to a random location on each boot, making writing exploits that depend on local resources significantly harder.

The side effect of this feature is debugging tools like crash may encounter some trouble trying to open vmcores from KASLR-enabled kernels.

For more information about this, please refer to this article.
```

可以参考这里:
* https://access.redhat.com/articles/3402021

但它也提到，`crash`可以直接读取VMware的快照文件，步骤如下:

* 复制`ESXi`主机上`Guest`所在目录下的`.vmss`/`.vmsn`和`.vmem`文件到一个目录
* 保持文件名称不变
* 使用`crash`工具打开`.vmsn`/`.vmss`文件，`crash`会自动加载相应的`.vmem`文件

如:
```bash
# crash <NAME>.vmsn  /cores/retrace/repos/kernel/x86_64/usr/lib/debug/lib/modules/<kernel version>/vmlinux
```

不过在生成快照时，需要注意勾选`包括虚拟机的内存`:
{% img /images/2022-07-18/2.png %}

有时侯系统可能会连`NMI`也无法响应，这种场景下就只能使用快照方式。但快照方式本身需要从`ESXi`主机上获取快照文件，也不是很方便。总之各有利弊，看情况选择使用哪种方法。

因为使用`crash`工具分析`dump`文件，需要对应内核版本的`debuginfo`包。有时我们拿到`dump`文件，却不知道内核版本。我们可以从`dump`文件中直接查找字符串来搜索, 如:
```bash
[root@localhost centos76-Snapshot1]# strings vmss.core  |grep -i 'BOOT_IMAGE'
BOOT_IMAGE=/vmlinuz-3.10.0-957.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet LANG=en_US.UTF-8
$(cat /proc/cmdline| grep "BOOT_IMAGE" | cut -d' ' -f1)
KDUMP_BOOTDIR="/boot"$(dirname $BOOT_IMAGE)
BOOT_IMAGE
```

#### 参考:
* https://support.delphix.com/Continuous_Data_Engine_(formerly_Virtualization_Engine)/Platforms/How_to_Generate_a_non-maskable_interrupt_in_VMware_ESX__(KBA1129)
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-generic_commands-injecting_nmi
* https://kb.vmware.com/s/article/1015180
* https://kb.vmware.com/s/article/2149185
* https://kb.vmware.com/s/article/67438
* https://kb.vmware.com/s/article/2003941
* https://flings.vmware.com/vmss2core
* https://access.redhat.com/articles/3587631
* https://access.redhat.com/solutions/210063
* https://access.redhat.com/solutions/3622951
* https://knowledge.broadcom.com/external/article/181598/how-to-convert-a-vmware-virtual-machine.html
