---
layout: post
title: "VMware vSphere平台下API操作虚拟交换机及虚拟接口"
date: 2017-12-19 10:59:25 +0800
comments: true
categories: Virtualization
---
之前的文章<<[VMware vSphere东西向网络防护](http://www.just4coding.com/blog/2017/06/10/vmware-westeast/)>>介绍了在`VMware vSphere`平台上如何通过操作虚拟交换机及虚拟接口来实现二层网络的微隔离。本文通过代码实例来说明调用API实现其中涉及的相关操作。

我们使用`VMware`官方的Python SDK来实现，SDK地址如下:

* https://github.com/vmware/pyvmomi

首先使用`pip`安装`pyvmomi`, 这里我们使用支持`vSphere6.5`的版本:
```bash
pip install pyvmomi==6.5.0.2017.5-1
```

下面介绍使用Python SDK的基本流程。

<!--more-->

使用该SDK，需要在Python中`import`该库:

```python
import pyVim
```

第一步，你需要连接到`ESXi`主机或者`vCenter`, 通常情况下，`vSphere`环境下使用`443`端口，如:

```python
from pyVim import connect
my_cluster = connect.Connect(“10.0.0.99”, 443, “username”, “password”)
```

处理完逻辑后需要关闭连接:

```python
connect.Disconnect(my_cluster)
```

连接建立后，可以查询虚拟机、获取虚拟机信息，发送命令等等。为了获取一个虚拟机对象，可以使用`searchIndex`类，该类可以通过`UUID`, `DNS名`，`IP地址`或者`datastore`的路径来查找虚拟机， 比如，下面示例会输出IP为`10.0.0.240`的虚拟机的名称及`UUID`:

```python
from pyVim import connect

my_cluster = connect.Connect(“10.0.0.99", 443, “username", “password")

searcher = my_cluster.content.searchIndex

vm = searcher.FindByIp(ip="10.0.0.240", vmSearch=True)
print vm.config.name
print vm.config.uuid

connect.Disconnect(my_cluster)
```

执行后结果如下:

```python
(pyvmomi)[root@centos1 fg]# python t.py
VC6.5
564d71d4-709c-f475-d255-0b695d071bd3
```

下面直接以代码示例来说明对于虚拟交换机及虚拟接口的操作。

创建虚拟交换机:

```python
from pyVim import connect
from pyVmomi import vim

my_cluster = connect.Connect("10.0.0.99", 443, "username", "password")

searcher = my_cluster.content.searchIndex

host = searcher.FindByIp(ip="10.0.0.41", vmSearch=False)
if not host:
    print "Host Not Found"
    exit(-1)

vswitch_spec = vim.host.VirtualSwitch.Specification()
vswitch_spec.numPorts = 1024
vswitch_spec.mtu = 1450
host.configManager.networkSystem.AddVirtualSwitch("vswitch_internal", vswitch_spec)

connect.Disconnect(my_cluster)
```

示例中，首先建立到`vCenter`的连接，接着查询到IP为`10.0.0.41`的宿主机, 在宿主机上创建一个名为`vswitch_internal`的标准虚拟交换机，最后关闭连接, 结果如图:

{% img /images/2017-12-19/1.png %}

添加端口组：

```python
from pyVim import connect
from pyVmomi import vim

my_cluster = connect.Connect("10.0.0.99", 443, "username", "password")

searcher = my_cluster.content.searchIndex

host = searcher.FindByIp(ip="10.0.0.41", vmSearch=False)
if not host:
    print "Host Not Found"
    exit(-1)

portgroup_spec = vim.host.PortGroup.Specification()
portgroup_spec.vswitchName = "vswitch_internal"
portgroup_spec.name = "vlan1000"
portgroup_spec.vlanId = 1000
network_policy = vim.host.NetworkPolicy()
network_policy.security = vim.host.NetworkPolicy.SecurityPolicy()
network_policy.security.allowPromiscuous = True
network_policy.security.macChanges = True
network_policy.security.forgedTransmits = True
portgroup_spec.policy = network_policy

host.configManager.networkSystem.AddPortGroup(portgroup_spec)

connect.Disconnect(my_cluster)
```

在以上示例中，首先建立到`vCenter`的连接，接着查询到IP为`10.0.0.41`的宿主机，在该宿主机上名为`vswitch_internal`的虚拟交换机上添加了一个`VLAN TAG`为`1000`，名称为`vlan1000`的端口组，并将三个安全选项都设置为接受，最后关闭连接，结果如图:

{% img /images/2017-12-19/2.png %}

修改虚拟机网卡所连接的端口组:

```python
from pyVim import connect
from pyVmomi import vim

my_cluster = connect.Connect("10.0.0.99", 443, "username", “password")

searcher = my_cluster.content.searchIndex
host = searcher.FindByIp(ip="10.0.0.41", vmSearch=False)
if not host:
    print("Host Not Found")
    exit(-1)

vm = searcher.FindByIp(ip="10.0.0.240", vmSearch=True)
if not vm:
    print("VM Not Found")
    exit(-1)

network = None
for n in host.network:
    if n.name == "vlan1000":
        network = n
        break

if not network:
    print("Network Not Found")
    exit(-1)

device_change = []
for device in vm.config.hardware.device:
    if isinstance(device, vim.vm.device.VirtualEthernetCard):
        nicspec = vim.vm.device.VirtualDeviceSpec()
        nicspec.operation = \
            vim.vm.device.VirtualDeviceSpec.Operation.edit
        nicspec.device = device
        nicspec.device.wakeOnLanEnabled = True

        nicspec.device.backing = \
vim.vm.device.VirtualEthernetCard.NetworkBackingInfo()
nicspec.device.backing.network = network
        nicspec.device.backing.deviceName = "vlan1000"

        nicspec.device.connectable = \
            vim.vm.device.VirtualDevice.ConnectInfo()
        nicspec.device.connectable.startConnected = True
        nicspec.device.connectable.allowGuestControl = True
        device_change.append(nicspec)
        break

config_spec = vim.vm.ConfigSpec(deviceChange=device_change)
try:
    vm.ReconfigVM_Task(config_spec)
except Exception, e:
    print(str(e))
    exit(-1)

connect.Disconnect(my_cluster)
```

本示例将IP为`10.0.0.240`的虚拟机的虚拟网卡修改到上面创建的`vlan1000`端口组，结果如图:

{% img /images/2017-12-19/3.png %}

上述示例尽量做了简化，若需要在生产环境使用时，可以参考官方示例库及相应文档:

* http://vmware.github.io/pyvmomi-community-samples/
* https://github.com/vmware/pyvmomi/tree/master/docs
