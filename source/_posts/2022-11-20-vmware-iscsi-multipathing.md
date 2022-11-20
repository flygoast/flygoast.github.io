---
title: VMware vSphere环境iSCSI配置多路径
date: 2022-11-20 18:02:45
tags:
- VMware
- iSCSI
categories: Virtualization
---
`VMware vSphere`的很多高级特性都依赖于共享存储, 如`vMotion`, `HA: High Availability`, `DRS: Distributed Resource Schedule`等, 它们要生效都需要虚拟机的存储位于共享存储中. `vSphere`支持的共享存储除了自家的`vSAN`, 还包括: `NFS`, `iSCSI`, `光纤通道: Fibre Channel`等. 

`iSCSI`是一个标准协议, 全称为:`Internet Small Computer System Interface`, 它在`以太网`上基于`TCP/IP`协议来传输`SCSI`协议. `SCSI`协议是计算机上的`I/O`传输协议, `SCSI`控制器通过`SCSI`总线与硬盘等设备以块为单位传输数据. `iSCSI`服务器称为`target`, 客户端称为`initiator`. `iSCSI initiator`能够以纯软件实现运行在标准网络适配器上, 也可以以硬件形式实现为专用的`HBA`卡:(`Host Bus Adaptor`), 也有带有`iSCSCI offload`硬件支持的网卡可以来加速`iSCSI`协议处理. 

`iSCSI`协议层次如图(来自: https://www.snia.org/education/what-is-iscsi):
{% img /images/2022-11-20/1.png %}

<!--more-->

几乎所有的企业级存储产品都支持`iSCSI`协议. 在`Windows`和`Linux`等操作系统上也都有成熟稳定的`iSCSI`实现. 在`CentOS`上配置`iSCSI`也非常简单, 有大量的文章介绍, 可以参考:

* https://www.thegeekdiary.com/complete-guide-to-configuring-iscsi-in-centos-rhel-7/
* https://www.centlinux.com/2018/08/configure-iscsi-target-initiator-centos-7.html
* https://www.tecmint.com/create-centralized-secure-storage-using-iscsi-targetin-linux/


由于`iSCSI`存储数据是基于网络传输, 为了提升性能, 可以采用`多路径: Multipathing`机制来传输数据, 其中的一种实现方式是一个`iSCSI session`使用多个连接, 叫做`MC/S: Multiple Connections per Session`. 

`VMware vSphere`支持`iSCSI`协议. 介绍在`vSphere`上配置`iSCSI`的文章也非常多, 可以参考:

* https://4sysops.com/archives/how-to-configure-esxi-6-5-for-iscsi-shared-storage/
* https://xpertstec.com/how-to-connect-iscsi-storage-vcenter-7-openfiler/
* http://woshub.com/add-iscsi-datastore-lun-vmware-esxi/
* https://www.unixmen.com/how-to-create-an-iscsi-target/
* https://stonywall.com/2021/01/15/setting-up-iscsi-on-esxi-7-with-vcenter-7/
* https://kb.synology.com/en-nz/DSM/tutorial/How_to_Use_iSCSI_Targets_on_VMware_ESXi_Server_with_Multipath_Support


`VMware vSphere`的`iSCSI initiator`实现也支持`Multipathing`, 是基于`端口绑定: Port Binding`实现, 如图:
{% img /images/2022-11-20/2.png %}

一般, 在`vSphere`上配置`iSCSI`时, 会建立专用的`VMkernel`接口和网络端口组. 但需要注意的是用于`multipathing`的端口组的`uplink`接口, 只能有一个`活动上行链路`, 并且没有`备用上行链路`. 如果虚拟交换机维度存在`备用上行链路`, 在`iSCSI`端口组上需要将该链路移动`未使用的上行链路`. 如图:
{% img /images/2022-11-20/3.png %}

为了实现`multipathing`, 需要至少建立两个这样的`VMkernel`接口和网络端口组, 分别指定不同的上行链路, 然后全部添加到`iSCSI software adaptor`的`端口绑定`中, 如图:
{% img /images/2022-11-20/4.png %}

可以参考:

* https://kb.vmware.com/s/article/2045040


但是, 实际使用`Port binding`时是有一些注意事项的, 请参考[`VMware`官方KB](https://kb.vmware.com/s/article/2038869?lang=zh_cn)

`Port binding`本质上是指定使用哪个`VMKernel`建立`iSCSI`网络连接. 当多个`VMKernel`接口位于相同广播域时, 默认`ESXi`只会使用第一个接口的路由表, 当需要使用多个连接时, 需要配置`Port binding`.

参考链接里说, 当多个`VMKernel`位于不同子网时, 不需要配置`Port binding`, 但个人感觉, 如果想使用`load balance`模式去利用多条连接而不是`fail over`模式, 也是需要配置的. 后续有时间可以再实验一下.

参考:

* http://www.open-iscsi.com/
* https://dusays.com/304/
* https://www.snia.org/education/what-is-iscsi
* https://datatracker.ietf.org/doc/html/rfc3720
* https://datatracker.ietf.org/doc/html/rfc7143
* https://www.thomas-krenn.com/en/wiki/ISCSI_Basics
* https://infohub.delltechnologies.com/l/iscsi-implementation-guide-for-dell-emc-storage-arrays-running-powermaxos-1/introduction-to-iscsi
* https://www.enterprisestorageforum.com/hardware/what-is-iscsi-and-how-does-it-work/
* https://www.thegeekdiary.com/complete-guide-to-configuring-iscsi-in-centos-rhel-7/
* https://computingforgeeks.com/configure-iscsi-target-and-initiator-on-centos-rhel/
* https://www.centlinux.com/2018/08/configure-iscsi-target-initiator-centos-7.html
* https://www.tecmint.com/create-centralized-secure-storage-using-iscsi-targetin-linux/
* https://xpertstec.com/how-to-connect-iscsi-storage-vcenter-7-openfiler/
* http://woshub.com/add-iscsi-datastore-lun-vmware-esxi/
* https://www.unixmen.com/how-to-create-an-iscsi-target/
* https://stonywall.com/2021/01/15/setting-up-iscsi-on-esxi-7-with-vcenter-7/
* https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.storage.doc/GUID-DD2FFAA7-796E-414C-84CE-1FCC14474D5B.html#GUID-DD2FFAA7-796E-414C-84CE-1FCC14474D5B
* https://subscription.packtpub.com/book/cloud-and-networking/9781789953008/5/ch05lvl1sec43/iscsi-multipathing-using-port-binding
* https://www.hiroom2.com/2017/08/04/centos-7-iscsi-initiator-utils-en/
* https://kb.synology.com/en-nz/DSM/tutorial/How_to_Use_iSCSI_Targets_on_VMware_ESXi_Server_with_Multipath_Support
* https://kb.vmware.com/s/article/2010877
* https://kb.vmware.com/s/article/2038869?lang=zh_cn
* https://www.stephenwagner.com/2014/06/07/vmware-vsphere-iscsi-port-binding/
* https://www.stephenwagner.com/2014/06/07/vsphere-distributed-switch-configuration-for-iscsi-mpio-san-using-multiple-subnets/
* https://www.codyhosterman.com/2015/03/esxi-iscsi-multipathing/
