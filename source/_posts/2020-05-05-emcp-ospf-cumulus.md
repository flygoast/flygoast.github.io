---
title: 基于Cumulus VX实验ECMP+OSPF负载均衡
date: 2020-05-05 09:16:30
tags:
- EMCP
- OSPF
- Cumulus VX
- LoadBalance
categories: Network
---
互联网服务为了保证高可用和可扩展性，在流量入口一般都需要部署负载均衡设备。负载均衡设备可分为4层(传输层)和7层(应用层)。L4负载均衡设备之前较为流行的方案是`LVS`，后来各大厂商又基于`DPDK`或`XDP`/`eBPF`等技术实现了性能更高的一些方案, 如:
* Google: [Maglev](https://research.google/pubs/pub44824.pdf)
* Github: [GLB](https://github.com/github/glb-director)
* Facebook: [Katran](https://github.com/facebookincubator/katran)

无论以上哪种实现，流量分发的部署方案都类似, 都是由LB集群中多个服务器通过`BGP`或者`OSPF`协议向网络设备宣告相同的IP地址(VIP)，网络设备通过`ECMP`路由将流量分散到这些服务器中。当其中某些LB服务器异常宕机或者维护下线时，网络设备会通过`OSPF`或者`BGP`协议的路由收敛机制检测到LB服务器下线，迅速将流量切走, 实现服务高可用。

本文来实验基于`ECMP`和`OSPF`的负载均衡。[Cumulus](https://cumulusnetworks.com/)是一个基于Linux的网络操作系统(NOS:Network Operation System), 运行于白牌交换机上。[Cumulus VX](https://cumulusnetworks.com/products/cumulus-vx/)是Cumulus的免费虚拟设备，可以以虚拟机形式运行。本文就使用Vagrant, VirtualBox, Ubuntu, Cumulus VX来构建我们的实验拓扑。结构如图:

{% img /images/2020-05-05/1.png %}

<!--more-->

路由器`router`的接口`swp1`做为客户端网段的网关`172.16.100.1`, 接口`swp2`做为负载均衡器网段的网关`192.168.100.1`。在`router`上开启`OSPF`，`lb1`和`lb2`上分别运行[FRR](https://frrouting.org/)使用`OSPF`协议向`router`来宣告`VIP`。当请求到达`router`时，由`router`根据学习到的两条路由基于`ECMP`的内部HASH机制计算路径，将请求分散转发到`lb1`和`lb2`。

基于`Cumulus VX`、`Vagrant`、`VirtualBox`来构建实验环境，可以参考:
https://www.actualtech.io/tutorial-deploying-a-cumulus-linux-demo-environment-with-vagrant/

Cumulus官方还提供了一个程序[topology_convert](https://gitlab.com/cumulus-consulting/tools/topology_converter/)可以自动生成`Vagrantfile`文件。

这里我们直接手动编写`Vagrantfile`:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # router
  config.vm.define "router" do |device|
    device.vm.hostname = "router"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    device.vm.network "private_network", virtualbox__intnet: "intnet-rt-sw1", auto_config: false
    device.vm.network "private_network", virtualbox__intnet: "intnet-rt-sw2", auto_config: false
    device.vm.provision :shell, privileged: true, :inline => 'ip route del default'
  end

  # sw1
  config.vm.define "sw1" do |device|
    device.vm.hostname = "sw1"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end

    device.vm.network "private_network", virtualbox__intnet: "intnet-rt-sw1", auto_config: false
    device.vm.network "private_network", virtualbox__intnet: "intnet-sw1-cli", auto_config: false
    device.vm.provision :shell, privileged: true, :inline => 'ip route del default'
  end

  # sw1
  config.vm.define "sw2" do |device|
    device.vm.hostname = "sw2"
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end

    device.vm.network "private_network", virtualbox__intnet: "intnet-rt-sw2", auto_config: false
    device.vm.network "private_network", virtualbox__intnet: "intnet-sw2-lb1", auto_config: false
    device.vm.network "private_network", virtualbox__intnet: "intnet-sw2-lb2", auto_config: false
    device.vm.provision :shell, privileged: true, :inline => 'ip route del default'
  end

  # lb1 
  config.vm.define "lb1" do |device|
    device.vm.hostname = "lb1"
    device.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 512
    end

    device.vm.box = "ubuntu/bionic64"
    device.vm.network "private_network", virtualbox__intnet: "intnet-sw2-lb1", auto_config: false
  end

  # lb2 
  config.vm.define "lb2" do |device|
    device.vm.hostname = "lb2"
    device.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 512
    end

    device.vm.box = "ubuntu/bionic64"
    device.vm.network "private_network", virtualbox__intnet: "intnet-sw2-lb2", auto_config: false
  end

  # cli 
  config.vm.define "cli" do |device|
    device.vm.hostname = "cli"
    device.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 512
    end

    device.vm.box = "ubuntu/bionic64"
    device.vm.network "private_network", virtualbox__intnet: "intnet-sw1-cli", auto_config: false
  end
end
```

构建环境:
```bash
vagrant up
```

环境构建完成后，分别在`lb1`和`lb2`两台服务器上安装`FRR`和`nginx`:
```bash
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
FRRVER="frr-stable"
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | sudo tee -a /etc/apt/sources.list.d/frr.list
sudo apt update && sudo apt install frr frr-pythontools
sudo apt-get install nginx
```

我们最终使用HTTP请求来测试负载均衡效果，为了便于在客户端识别HTTP响应来自于哪台LB，修改NGINX的默认配置文件`/etc/nginx/sites-enabled/default`, 将`location /`的处理逻辑分别修改为:
```plain
return 200 "lb1\n";
```

```plain
return 200 "lb2\n";
```

配置`lb1`的IP:
```bash
ip addr add 192.168.100.2/24 dev enp0s8
ip link set up enp0s8
ip route del default
ip route add default via 192.168.100.1
```

配置`lb2`的IP:
```bash
ip addr add 192.168.100.3/24 dev enp0s8
ip link set up enp0s8
ip route del default
ip route add default via 192.168.100.1
```
在`lb2`上访问`lb1`, 此时访问不通，因为二层交换机`sw2`还没有配置。
```plain
root@lb2:/home/vagrant# ping -c2 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.

--- 192.168.100.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1021ms
```

登录到`sw2`上配置二层转发交换机, 这里我们使用Cumulus提供的命令行工具`NCLU(Network Command Line Utility)`:
```plain
vagrant@sw2:~$ sudo su - cumulus
cumulus@sw2:~$ net add bridge br1 ports swp1-3
cumulus@sw2:~$ net commit
```

查看接口情况:
```plain
root@sw2:~# net show interface
State  Name  Spd  MTU    Mode       LLDP  Summary
-----  ----  ---  -----  ---------  ----  ----------------------
UP     lo    N/A  65536  Loopback         IP: 127.0.0.1/8
       lo                                 IP: ::1/128
UP     eth0  1G   1500   Mgmt             IP: 10.0.2.15/24(DHCP)
UP     swp1  1G   1500   Access/L2        Master: br1(UP)
UP     swp2  1G   1500   Access/L2        Master: br1(UP)
UP     swp3  1G   1500   Access/L2        Master: br1(UP)
UP     br1   N/A  1500   Bridge/L2
```

此时再次从`lb2`访问`lb1`, 访问成功:
```plain
root@lb2:/home/vagrant# ping -c2 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=2.25 ms
64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=1.39 ms

--- 192.168.100.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.398/1.825/2.253/0.429 ms
```

接下来我们配置`router`:
```plain
cumulus@router:~$ net add interface swp1 ip address 172.16.100.1/24
cumulus@router:~$ net add interface swp2 ip address 192.168.100.1/24
cumulus@router:~$ net commit
```

查看接口情况:
```plain
cumulus@router:~$ net show interface
State  Name  Spd  MTU    Mode          LLDP        Summary
-----  ----  ---  -----  ------------  ----------  ----------------------
UP     lo    N/A  65536  Loopback                  IP: 127.0.0.1/8
       lo                                          IP: ::1/128
UP     eth0  1G   1500   Mgmt                      IP: 10.0.2.15/24(DHCP)
UP     swp1  1G   1500   Interface/L3              IP: 172.16.100.1/24
UP     swp2  1G   1500   Interface/L3  sw2 (swp1)  IP: 192.168.100.1/24
```

配置二层交换机`sw1`:
```plain
root@sw1:~# net add bridge br1 ports swp1-2
root@sw1:~# net commit
```

查看接口:
```plain
root@sw1:~# net show interface
State  Name  Spd  MTU    Mode       LLDP           Summary
-----  ----  ---  -----  ---------  -------------  ----------------------
UP     lo    N/A  65536  Loopback                  IP: 127.0.0.1/8
       lo                                          IP: ::1/128
UP     eth0  1G   1500   Mgmt                      IP: 10.0.2.15/24(DHCP)
UP     swp1  1G   1500   Access/L2  router (swp1)  Master: br1(UP)
UP     swp2  1G   1500   Access/L2                 Master: br1(UP)
UP     br1   N/A  1500   Bridge/L2
```

配置客户端虚拟机`cli`:
```plain
ip addr add 172.16.100.2/24 dev enp0s8
ip link set up enp0s8
ip route del default
ip route add default via 172.16.100.1
```

此时从客户端`cli`访问两个LB的自身IP地址，访问都可以成功:
```plain
root@cli:/home/vagrant# ping -c2 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
64 bytes from 192.168.100.2: icmp_seq=1 ttl=63 time=2.85 ms
64 bytes from 192.168.100.2: icmp_seq=2 ttl=63 time=3.17 ms

--- 192.168.100.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.854/3.012/3.170/0.158 ms
```

```plain
root@cli:/home/vagrant# ping -c2 192.168.100.3
PING 192.168.100.3 (192.168.100.3) 56(84) bytes of data.
64 bytes from 192.168.100.3: icmp_seq=1 ttl=63 time=2.80 ms
64 bytes from 192.168.100.3: icmp_seq=2 ttl=63 time=3.81 ms

--- 192.168.100.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 2.806/3.308/3.811/0.505 ms
```

接下来我们在`router`上配置`OSPF`:
```plain
net add ospf router-id 192.168.100.1
net add ospf network 192.168.100.0/24 area 0.0.0.0
net commit
```
此时`opsfd`进程已经启动。

我们使用`10.10.10.0/24`做为`VIP`地址池, 接着在`lb1`和`lb2`两台LB机器上配置`OSPF`宣告VIP地址。

在`lb1`上给`lo`添加VIP地址:
```bash
ip addr add 10.10.10.10/32 dev lo
ip addr add 10.10.10.11/32 dev lo
ip addr add 10.10.10.12/32 dev lo
```

接着，修改`/etc/frr/daemons`文件, 确保`ospfd`配置开启:
```plain
ospfd=yes
```

创建`ospfd.conf`文件, 没有该文件`ospfd`进程不会启动:
```bash
touch /etc/frr/ospfd.conf
```

修改配置文件`/etc/frr/frr.conf`, 添加内容:
```plain
router ospf
  ospf router-id 192.168.100.2
  network 192.168.100.0/24 area 0.0.0.0
  network 10.10.10.0/24 area 0.0.0.0
!
interface enp0s8
  ip ospf area 0.0.0.0
!
```

启动`FRR`:
```bash
systemctl start frr
```

在`lb2`上也完成相应修改。


此时我们在`router`上查看`OSPF`路由信息, 可以看到3个`VIP`的路由信息都已学习到:
```plain
root@router:~# net show route ospf
RIB entry for ospf
==================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

O>* 10.10.10.10/32 [110/100] via 192.168.100.2, swp2, 00:24:08
  *                          via 192.168.100.3, swp2, 00:24:08
O>* 10.10.10.11/32 [110/100] via 192.168.100.2, swp2, 00:24:08
  *                          via 192.168.100.3, swp2, 00:24:08
O>* 10.10.10.12/32 [110/100] via 192.168.100.2, swp2, 00:24:08
  *                          via 192.168.100.3, swp2, 00:24:08
O   192.168.100.0/24 [110/100] is directly connected, swp2, 02:07:49
```

此时，我们在客户端`cli`上访问VIP:`10.10.10.10`, 发现访问并没有分布在两台LB机器上, 而是一直只访问一台LB机器。这是由于Linux内核的`ECMP`默认的HASH策略是使用`L3`信息，即源IP、目的IP。因为我们只在一台客户端上进行测试，因而HASH计算结果一直相同，只会一直落在一台LB机器上。

根据内核参数信息:
```plain
fib_multipath_hash_policy - INTEGER
	Controls which hash policy to use for multipath routes. Only valid
	for kernels built with CONFIG_IP_ROUTE_MULTIPATH enabled.
	Default: 0 (Layer 3)
	Possible values:
	0 - Layer 3
	1 - Layer 4
	2 - Layer 3 or inner Layer 3 if present
```

我们修改`fib_multipath_hash_policy`, 使`ECMP`基于`L4`信息进行HASH计算：
```bash
sysctl -w net.ipv4.fib_multipath_hash_policy=1
```

这时，再次从`cli`进行测试, 访问VIP`10.10.10.10`, 请求已经到达两台LB机器上了:
```plain
root@cli:/home/vagrant# curl 10.10.10.10
lb1
root@cli:/home/vagrant# curl 10.10.10.10
lb2
root@cli:/home/vagrant# curl 10.10.10.10
lb1
root@cli:/home/vagrant# curl 10.10.10.10
lb2
```

我们在`lb1`上关闭`FRR`来模拟LB机器下线或者宕机:
```bash
systemctl stop frr
```

此时，再次从`cli`机器上访问`10.10.10.10`, 所有请求都成功转发给`lb2`了:
```plain
root@cli:/home/vagrant# curl 10.10.10.10
lb2
root@cli:/home/vagrant# curl 10.10.10.10
lb2
root@cli:/home/vagrant# curl 10.10.10.10
lb2
root@cli:/home/vagrant# curl 10.10.10.10
lb2
root@cli:/home/vagrant# curl 10.10.10.10
lb2
root@cli:/home/vagrant# curl 10.10.10.10
lb2
```

在实际应用场景中，这种负载均衡方案不只可以应用在L4/L7负载均衡设备，对于无状态或者独立的应用集群可以应用，比如，权威DNS服务器集群也可以以这种方式来构建。

参考链接:
* https://codecave.cc/multipath-routing-in-linux-part-2.html
* https://www.theasciiconstruct.com/home/categories/cumulus
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
* https://cumulusnetworks.com/blog/celebrating-ecmp-part-two/
