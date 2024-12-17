---
title: 使用runc运行ansible
date: 2024-12-17 20:03:29
tags:
- Ansible
- runc
categories: Network
---
我们的系统使用`ansible`执行部署过程，然而部署的目标操作系统和CPU架构环境多种多样，存在大量适配工作。之前调研过是否有`Golang`开发的`ansible`替代物，但没有找到合适的方案。这种到处运行的场景很适合使用容器，但如果使用`docker`,  在不同操作系统上也需要准备不同的`docker`安装包，依然需要适配，因而也不是很满足。今天想到`runc`本身就是`Golang`开发的纯静态二进制，非常适合到处分发，因而尝试调研直接使用`runc`来做为运行时运行`ansible`容器。

在容器领域，[`OCI:Open Container Initiative`](https://opencontainers.org/)制定了两个规范，镜像规范: `image-spec`和运行时规范:`runtime-spec`。镜像规范规定了镜像文件的格式，而运行时规范规定了如何根据配置运行容器。它们之间通过`filesystem bundle`联系在一起，镜像可以通过工具转换成`filesystem bundle`, 运行时根据`filesystem bundle`运行容器进程。

<!--more-->

容器运行时一般分为`High-level`运行时和`low-level`运行时，`runc`是用来根据`runtime-spec`运行容器的命令行工具，它属于于低级运行时: `low-level runtime`.

`filesystem bundle`包括两部分: 

*   配置文件：指定了容器进程相关的参数，如`namespace`, `capabilities`, `mounts`，环境变量等。
*   `rootfs`: 容器进程运行的根文件系统所在目录

下面首先来生成`filesystem bundle`。

```text-plain
mkdir -p ansible/rootfs
cd ansible
docker pull alpine/ansible
docker export $(docker create alpine/ansible:latest) | tar -C rootfs/ -xvf -
```

查看`rootfs`目录:

```text-plain
[root@localhost ansible]# ls -l rootfs/
总用量 12
drwxr-xr-x  2 root root 4096 12月 14 11:10 bin
drwxr-xr-x  4 root root   43 12月 17 15:27 dev
drwxr-xr-x 20 root root 4096 12月 17 15:27 etc
drwxr-xr-x  2 root root    6 12月  5 20:17 home
drwxr-xr-x  6 root root  127 12月  5 20:17 lib
drwxr-xr-x  5 root root   44 12月  5 20:17 media
drwxr-xr-x  2 root root    6 12月  5 20:17 mnt
drwxr-xr-x  2 root root    6 12月  5 20:17 opt
dr-xr-xr-x  2 root root    6 12月  5 20:17 proc
drwx------  5 root root   69 12月 17 16:56 root
drwxr-xr-x  2 root root    6 12月  5 20:17 run
drwxr-xr-x  2 root root 4096 12月  5 20:17 sbin
drwxr-xr-x  2 root root    6 12月  5 20:17 srv
drwxr-xr-x  2 root root    6 12月  5 20:17 sys
drwxrwxrwt  2 root root    6 12月 17 17:21 tmp
drwxr-xr-x  8 root root   81 12月 14 11:10 usr
drwxr-xr-x 11 root root  137 12月  5 20:17 var
```

可以从`Github`下载指定架构的`runc`二进制文件, 通过`runc`生成配置文件:

```text-plain
runc spec
```

这时会生成`config.json`文件。

创建一个测试容器，`config.json`默认指定运行`sh`命令。在其中运行`ip`命令查看网络信息:

```text-plain
[root@localhost ansible]# runc run t1
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
/ #
```

可以看到容器里没有外部网络接口。因为我们需要运行`ansible`访问其他机器，因而它需要使用`宿主网络`模式，类似于`docker`中`host`网络模式。根据`runc`官方的这个`issue`: [https://github.com/opencontainers/runc/issues/1552](https://github.com/opencontainers/runc/issues/1552)，我们从`config.json`中`namespaces`对象中删除`network`后，再次运行测试容器:

```text-plain
[root@localhost ansible]# runc run test
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:81:dc:16 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.2/23 brd 10.10.10.255 scope global ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe81:dc16/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:37:ec:d9:5c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:37ff:feec:d95c/64 scope link
       valid_lft forever preferred_lft forever
21: vethfbb20cf@if20: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether 82:8f:a3:e6:2c:e0 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::808f:a3ff:fee6:2ce0/64 scope link
       valid_lft forever preferred_lft forever
23: vethe11b87e@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether 72:59:7c:5d:98:55 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::7059:7cff:fe5d:9855/64 scope link
       valid_lft forever preferred_lft forever
41: veth6998371@if40: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether c2:0b:27:c8:d9:6d brd ff:ff:ff:ff:ff:ff
    inet6 fe80::c00b:27ff:fec8:d96d/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到，容器进程已经是宿主网络。

因为`ansible`使用`SSH`远程登录执行，需要将`.ssh`目录映射到容器中。修改`config.json`添加上`.ssh`目录:

```text-plain
    "mounts": [
        {
            "destination": "/root/.ssh/",
            "type": "bind",
            "source": "/root/.ssh/",
            "options": [
                "rbind",
                "rprivate"
            ]
        },
        ...
    ]
```

确保在容器内可以`SSH`登录目标机器。

默认情况下，`runc`以只读权限挂载`rootfs`，在容器中执行`ansible`命令会报错: 

```text-plain
/ # ansible -v
ERROR: Unhandled exception when retrieving DEFAULT_LOCAL_TMP:
[Errno 30] Read-only file system: '/root/.ansible/tmp/ansible-local-13sluq165n'. [Errno 30] Read-only file system: '/root/.ansible/tmp/ansible-local-13sluq165n'
```

我们需要修改`config.json`，将`root`下的`readonly`改为`false`:

```text-plain
    "root": {
        "path": "rootfs",
        "readonly": false
    },
```

再次运行可以正常执行:

```text-plain
[root@localhost ansible]# runc run test
/ # ansible --version
ansible [core 2.18.1]
  config file = None
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.12/site-packages/ansible
  ansible collection location = /root/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.12.8 (main, Dec  7 2024, 05:56:13) [GCC 14.2.0] (/usr/bin/python3)
  jinja version = 3.1.4
  libyaml = True
/ #
```

在`/root/ansible/`目录下创建一个测试用的`inventory`文件, 测试`ansible`是否可以正常执行:

```text-plain
/ # ansible -i /root/ansible/inventory -m shell -a 'hostname' 10.10.10.5
10.10.10.5 | CHANGED | rc=0 >>
centos8
```

`ansible`正常执行成功。

上面的执行是通过交互式方式，实际的场景需要非交互式执行，再次修改`config.sh`, 将`terminal`修改为`false`, 这样可以创建后台容器:

```text-plain
    "process": {
        "terminal": false,
        ...
```

通过`create`命令一个容器:

```text-plain
[root@localhost ansible]# runc create ansible
[root@localhost ansible]# runc list
ID          PID         STATUS      BUNDLE                         CREATED                         OWNER
ansible     1461        created     /root/workspace/runc/ansible   2024-12-17T11:59:27.69669486Z   root
```

尝试使用`exec`子命令执行`ansible`命令:

```text-plain
[root@localhost ansible]# runc exec ansible ansible -i /root/ansible/inventory -m shell -a 'hostname' 10.10.10.5
10.10.10.5 | CHANGED | rc=0 >>
centos8
```

这样就可以一直使用这一个后台容器执行不同的`ansible`命令了。

为了方便进一步使用，可以在宿主机上通过`alias`简化命令:

```plain
alias ansible="runc exec ansible ansible"
```

再次执行:
```plain
[root@localhost ansible]# ansible -i /root/ansible/inventory -m shell -a 'hostname' 10.10.10.5
10.10.10.5 | CHANGED | rc=0 >>
centos8
```

参考:

*   [https://cizixs.com/2017/11/05/oci-and-runc/](https://cizixs.com/2017/11/05/oci-and-runc/)
*   [https://benjamintoll.com/2022/01/18/on-runc/#mounts](https://benjamintoll.com/2022/01/18/on-runc/#mounts)
*   [https://blog.quarkslab.com/digging-into-runtimes-runc.html](https://blog.quarkslab.com/digging-into-runtimes-runc.html)
*   [https://medium.com/@Mark.io/https-medium-com-mark-io-network-setup-with-runc-containers-46b5a9cc4c5b](https://medium.com/@Mark.io/https-medium-com-mark-io-network-setup-with-runc-containers-46b5a9cc4c5b)
*   [https://medium.com/@Mark.io/beginners-guide-to-runc-1b29cf281752](https://medium.com/@Mark.io/beginners-guide-to-runc-1b29cf281752)
