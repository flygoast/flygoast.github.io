---
layout: post
title: "VMware环境中根据虚拟机IP找寻所在ESXi主机"
date: 2017-10-12 16:31:43 +0800
comments: true
categories: Virtualization
---
在VMware vSphere虚拟环境中我们有时需要找寻某IP所在的虚拟机及ESXi宿主机。若VMware虚拟机安装了VMware tools, 则可以通过API直接查找该IP所在位置，但我们的环境中并不是所有的虚拟机都已安装，因而我们只能通过MAC地址来查找。

假设目标IP为`10.95.48.11`，首先我们从与目标IP位于相同二层网络内的虚拟机上获取`10.95.48.11`对应的MAC地址:

```plain
[root@localhost ~]# ping 10.95.48.11 -c 2
PING 10.95.48.11 (10.95.48.11) 56(84) bytes of data.
64 bytes from 10.95.48.11: icmp_seq=1 ttl=64 time=0.141 ms
64 bytes from 10.95.48.11: icmp_seq=2 ttl=64 time=0.137 ms

--- 10.95.48.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.137/0.139/0.141/0.002 ms
[root@localhost ~]# ip neighbor |grep 10.95.48.11
10.95.48.11 dev eth0 lladdr 00:0c:29:26:18:c8 REACHABLE
```

获取到MAC地址为`00:0c:29:26:18:c8`。

<!--more-->

若是环境中ESXi主机较少，可以通过SSH直接登录ESXi主机挨个进行查找。

首先进入虚拟机的存储目录，不同环境中数据存储的名字可能不同:
```bash
cd /vmfs/volumes/datastore1/
```

该目录存储的是各个虚拟机的相关文件，一个虚拟机对应一个目录，如:
```bash
[root@esxi-01:/vmfs/volumes/595b7497-d8849df8-8d7c-6c92bf585d10] ls -l
total 176
drwxr-xr-x    1 root     root           420 Sep 14 02:30 centos-68
drwxr-xr-x    1 root     root          1820 Oct  9 09:55 dev01-10.95.48.11
drwxr-xr-x    1 root     root          3080 Sep 14 03:37 dev02-10.95.48.12
```

每台虚拟机目录中的`vmx`文件中存储了系统为虚拟网卡生成的MAC地址，如:

```plain
ethernet0.generatedAddress = "00:0c:29:26:18:c8"
ethernet0.generatedAddressOffset = "0"
```

我们可以从`vmx`文件中搜索MAC地址，找到相应的虚拟机，如:

```plain
[root@esxi-01:/vmfs/volumes/595b7497-d8849df8-8d7c-6c92bf585d10] find . -name '*.vmx' | xargs grep '00:0c:29:26:18:c8'
./dev01-10.95.48.11/dev01-10.95.48.11.vmx:ethernet0.generatedAddress = "00:0c:29:26:18:c8”
```

若是环境中ESXi主机非常多，一台一台搜索非常低效，我们可以基于VMware官方提供的SDK来编写程序来找到相应的MAC地址。

VMware提供了Python的SDK: https://github.com/vmware/pyvmomi

我们编写的程序如下:

```python
#!/usr/bin/env python
import atexit

from pyVim import connect
from pyVmomi import vmodl
from pyVmomi import vim

def print_vm_info(virtual_machine):
    for device in virtual_machine.config.hardware.device:
        if (device.key >= 4000) and (device.key < 5000):
            if device.macAddress == '00:0c:29:26:18:c8':
                print('device.macAddress==', device.macAddress)

                summary = virtual_machine.summary
                print("Name       : ", summary.config.name)
                print("Template   : ", summary.config.template)
                print("Path       : ", summary.config.vmPathName)
                print("Guest      : ", summary.config.guestFullName)
                print("Host       : ", summary.runtime.host.name)

def main():
    try:
        service_instance = connect.SmartConnect(host="10.10.10.10",
                                                user="administrator@vsphere.local",
                                                pwd="123456",
                                                port=443)

        atexit.register(connect.Disconnect, service_instance)

        content = service_instance.RetrieveContent()

        container = content.rootFolder  # starting point to look into
        viewType = [vim.VirtualMachine]  # object types to look for
        recursive = True  # whether we should look into it recursively
        containerView = content.viewManager.CreateContainerView(
            container, viewType, recursive)

        children = containerView.view
        for child in children:
            print_vm_info(child)

    except vmodl.MethodFault as error:
        print("Caught vmodl fault : " + error.msg)
        return -1

    return 0

# Start program
if __name__ == "__main__":
    main()
```

虚拟机的设备key值位于4000-5000表示网络设备，我们在网络设备的属性中查找MAC信息。程序中的连接信息可以是ESXi主机信息，也可以是vCenter信息。直接连接vCenter则可以将环境中所有ESXi主机全部搜索完, 避免一台一台主机搜索。

程序的执行结果，如下:

```plain
[root@vagrant-centos65 samples]# python get_vm_from_mac.py 
('device.macAddress==', '00:0c:29:26:18:c8')
('Name       : ', 'dev01-10.95.48.11')
('Template   : ', False)
('Path       : ', '[datastore1] dev01-10.95.48.11/dev01-10.95.48.11.vmx')
('Guest      : ', 'CentOS 4/5/6/7 (64-bit)')
('Host       : ', ‘esxi-01’)
```

