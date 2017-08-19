---
layout: post
title: "OpenStack私有网络安全防护"
date: 2017-08-19 21:42:33 +0800
comments: true
categories: OpenStack
---
之前的文章<<[VMware vSphere虚拟网络防护](http://www.just4coding.com/blog/2017/05/01/vmvare/)>>介绍了vSphere环境下基于NAT实现不同子网之间的网络安全防护。在OpenStack环境下网络安全防护也可以采用同样的方式。比如，我们在OpenStack环境中，有两个私有网络, 名称分别为`web`和`db`, 两个私有网络由虚拟路由连接。架构示意图如下:

{% img /images/2017-08-19/1.png %}

<!--more-->

为了防护私有网络`db`，可以将虚拟防火墙设备串接在虚拟路由器和私有网络`db`之间。需要访问私有网络`db`下的实例时，改为访问虚拟防火墙的IP，虚拟防火墙对数据包做NAT操作并完成安全过滤后，将数据包转发至DB服务器。架构示意图如下:

{% img /images/2017-08-19/2.png %}

图中的私有网络`fw-man`为虚拟防火墙实例的管理网络。OpenStack平台的网络拓朴图如下:

{% img /images/2017-08-19/3.png %}

在我们设置的环境中，所有私有网络`db`下实例的默认网关为`192.168.0.1`，因而我们在创建虚拟防火墙实例时，需要将连接私有网络db的接口IP设置为`192.168.0.1`，保证私有网络`db`下实例对外的三层流量都会发送至`fw`实例。

Horizon界面上不能自己指定接口IP，因而我们需要使用命令行指定IP为`192.168.0.1`创建好相应port，再attach到`fw`实例上:
```bash
neutron port-create 9789bff6-a502-4fe2-9baa-4446de37a43e --fixed-ip subnet_id=4f04cf0e-4229-46d7-b967-7ffa78a01d4b,ip_address=192.168.0.1
nova interface-attach --port-id 1a8224cf-a81a-4a78-b0ad-a828bd4e5802 fw
```
假设`web`虚拟机若需要访问`192.168.0.7:80`，我们在`fw`实例上配置DNAT规则，令`web`虚拟机访问`10.0.1.11:80`时将数据包转发至`192.168.0.7:80`:
```bash
iptables -t nat -A PREROUTING -d 10.0.1.11 --protocol tcp --dport 80 -j DNAT --to-destination 192.168.0.7:80
```
为了能够令数据包通过`fw`实例，需要开启数据转发功能:
```bash
sysctl -w net.ipv4.ip_forward=1
```
OpenStack有Anti Mac-Spoofing的机制。默认情况下，只有数据包的MAC、IP与当前接口所配置的MAC、IP都匹配时才能通过。`fw`实例中完成NAT操作后的数据包，由于IP地址已经更改会被安全规则丢弃。我们需要给`fw`实例上连接db网络的端口设置上可以通过的可用地址对(MAC、IP)。我们将可用地址对的IP设置为`0.0.0.0/0`, MAC设置为接口本身的MAC则允许所有数据包通过。

此外，由于fw实例上有多个连接外网的接口，需要对默认路由进行修改。这里我们手动指定`10.0.0.0/24`网段由网卡`eth0`发送。
```bash
ip route add 10.0.0.0/24 dev eth0 via 10.0.1.1
```

至此私有网络`web`到私有网络`db`的网络路径已经打通，只需要在`fw`实例上添加安全过滤功能就能完成网络安全防护。

这种方式下，虚拟安全设备实例与业务虚拟机实例共享底层的计算资源池。为了防止虚拟安全设备实例的资源消耗影响业务虚拟实例，特定场景下希望虚拟安全设备处于独立的安全资源池中。这种方式下则需要将流量牵引至独立的安全资源池。下面介绍思路及demo实现。

上述NAT方案的基本逻辑是数据包从一个网卡流入再从另一网卡流出。我们在这两个网卡的数据通道之间可以串接一个TAP设备。用户态进程从TAP设备读取数据包，封装后发送给外部的安全设备。安全设备过滤完再将数据封装后发送回用户态进程。用户态进程将数据包解封装后再从TAP设备发送给出口接口。

本文为了简单，AGENT直接将数据包以UDP负载发送，并以一个UDP服务器来直接将UDP负载中的数据包返回来模拟独立的安全资源池。架构示意图如下:

{% img /images/2017-08-19/4.png %}

环境创建过程如下：
```bash
ovs-vsctl add-br br0
ovs-vsctl add-port br0 eth1
ip addr del 192.168.0.1/24 dev eth1
ip addr add 192.168.0.1/24 dev br0
ip link set up br0
ip tuntap add dev tap0 mode tap
ip link set up tap0
ovs-vsctl add-port br0 tap0
```
此时OVS网桥结构如下:
```plain
[root@fw ~]# ovs-ofctl show br0
OFPT_FEATURES_REPLY (xid=0x2): dpid:00003a21b7bf9a4c
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 5(eth1): addr:fa:16:3e:3c:8c:b3
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 8(tap0): addr:ca:d8:6d:24:e0:8c
     config:     0
     state:      LINK_DOWN
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
 LOCAL(br0): addr:3a:21:b7:bf:9a:4c
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```
这样构造环境后，数据包从`eth0`到`eth1`的数据路径变更为”eth0->br0->eth1”。我们使用OpenFlow流表将TAP设备`tap0`串接入`br0`与`eth1`之间。

添加流表项用于正常转发ARP请求:
```bash
ovs-ofctl add-flow br0 "priority=10,dl_type=0x0806,in_port=LOCAL actions=output:5"
ovs-ofctl add-flow br0 "priority=10,dl_type=0x0806,in_port=5 actions=output:LOCAL”
```

添加流表项用于正常转发`fw`实例自身访问目标虚拟机的流量:
```bash
ovs-ofctl add-flow br0 "priority=9,dl_type=0x0800,nw_src=192.168.0.1/32, in_port=LOCAL, actions=output:5"
ovs-ofctl add-flow br0 "priority=9,dl_type=0x0800,nw_dst=192.168.0.1/32, in_port=5, actions=output:LOCAL”
```
添加流表项将从`br0`或者`eth1`流入的数据包转发到`tap0`:
```bash
ovs-ofctl add-flow br0 "priority=8,dl_type=0x0800,in_port=LOCAL actions=output:8"
ovs-ofctl add-flow br0 "priority=8,dl_type=0x0800,in_port=5 actions=output:8”
```
添加流表项根据`tap0`流入的数据包中的目的地址转发到`br0`或者`eth1`:
```bash
ovs-ofctl add-flow br0 "priority=7,dl_type=0x0800,nw_dst=192.168.0.0/24, in_port=8, actions=output:5"
ovs-ofctl add-flow br0 "priority=7,dl_type=0x0800,nw_dst=10.0.0.0/24, in_port=8, actions=output:LOCAL”
```
添加流表项将其他数据包丢弃:
```bash
ovs-ofctl add-flow br0 "priority=0, actions=drop”
```
`FW`实例的环境构造完成。

Agent代码如下:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <assert.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <net/if.h>
#include <linux/if_tun.h>


int udp_socket(char *remote_addr, int port)
{
    int                 fd;
    int                 length;
    struct sockaddr_in  skaddr;
    struct sockaddr_in  remote;

    assert((fd = socket( PF_INET, SOCK_DGRAM, 0 )) >= 0);

    skaddr.sin_family = AF_INET;
    skaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    skaddr.sin_port = htons(0);

    assert(bind(fd, (struct sockaddr *) &skaddr, sizeof(skaddr)) == 0);

    remote.sin_family = AF_INET;
    remote.sin_addr.s_addr = inet_addr(remote_addr);
    remote.sin_port = htons(port);
    assert(connect(fd, (struct sockaddr *) &skaddr, sizeof(remote)) == 0);

    return fd;
}


int open_tap(char *dev)
{
    struct ifreq  ifr;
    int  fd, err;

    assert(dev && *dev);
    assert((fd = open("/dev/net/tun", O_RDWR)) >= 0);
    memset(&ifr, 0, sizeof(ifr));

    ifr.ifr_flags = IFF_TAP;
    strncpy(ifr.ifr_name, dev, IFNAMSIZ);

    assert(ioctl(fd, TUNSETIFF, (void *) &ifr) >= 0);
    assert(strcmp(dev, ifr.ifr_name) == 0);

    return fd;
}


int main(int argc, char **argv)
{
    int     tap_fd;
    int     udp_fd;
    int     max_fd;
    int     ret;
    int     n;
    int     len;
    fd_set  rd_set;
    char    buf[4096];

    assert((tap_fd = open_tap("tap0")) >= 0);
    printf("TAP FD: %d\n", tap_fd);

    assert((udp_fd = udp_socket("10.95.46.177", 4789)) >= 0);
    printf("UDP FD: %d\n", udp_fd);

    max_fd = (tap_fd > udp_fd) ? tap_fd : udp_fd;

    while (1) {
        FD_ZERO(&rd_set);
        FD_SET(tap_fd, &rd_set);
        FD_SET(udp_fd, &rd_set);

        ret = select(max_fd + 1, &rd_set, NULL, NULL, NULL);

        if (ret < 0 && errno == EINTR) {
            continue;
        }

        if (ret < 0) {
            perror("select");
            exit(-1);
        }

        if (FD_ISSET(tap_fd, &rd_set)) {
            n = read(tap_fd, buf, sizeof(buf));
            assert(n >= 0);

            printf("READ from TAP: %d bytes\n", n);

            printf("WRITE to UDP\n");
            assert(write(udp_fd, buf, n) == n);
        }

        if (FD_ISSET(udp_fd, &rd_set)) {
            n = read(udp_fd, buf, sizeof(buf));
            assert(n >= 0);

            printf("READ from UDP: %d bytes\n", n);

            printf("WRITE to TAP\n");
            assert(write(tap_fd, buf, n) == n);
        }
    }

    exit(0);
}
```

将其编译运行:
```bash
gcc demo.c -o demo
./demo
```

UDP服务器的代码来自这里:
http://www.cs.rpi.edu/~goldsd/docs/spring2014-csci4220/echo-server-udp.c.txt

此时，`web`虚拟机实例访问`fw`实例的80端口，`fw`实例首先完成DNAT操作，数据包由`br0`接口进入OVS网桥。根据流表规则，数据包转发至TAP设备。AGENT从TAP设备读取数据包做为UDP负载发送至UDP服务器。UDP服务器将数据包返回给AGENT。AGENT将数据包写入TAP设备。数据包再次进入OVS网桥。根据流表规则，数据包转发至`eth1`, 从而送达`db`实例。响应包逻辑类似。数据包路径示意图如下:

{% img /images/2017-08-19/5.png %}

在实际方案中，数据包到达外部的安全资源池之后，安全资源池需要完成服务编排，令数据包流经一系列VNF安全设备，再流回`FW`实例。架构示意图如下:

{% img /images/2017-08-19/6.png %}

本文简要介绍了从私有网络内将流量牵引至云环境外部的实例，后续我们再来分析安全资源内的服务链实现。
