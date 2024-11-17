---
title: ctr和crictl命令的简单区别
date: 2024-11-17 14:14:30
tags:
- Kubernetes
categories: Kubernetes
---
[ctr](https://github.com/containerd/containerd/tree/98f48d485d25cf7a8b973c3f7477381727ae9f77/cmd/ctr)和[crictl](https://github.com/kubernetes-sigs/cri-tools)都是`Kubernetes`环境中管理容器的命令行工具。但它们的目的和使用方法有所不同。

`crictl`是基于`Kubernetes`的`CRI: Container Runtime Interface`接口规范来管理容器, `ctr`是containerd自带的容器管理工具, 本身和`Kubernetes`无关。

`Kubernetes`使用`crictl`来管理任意兼容`CRI`接口的容器运行时。

`containerd`相比于`docker`，增加了`namespace`的概念，每个`image`和`container`都在各自的`namespace`下可见。目前`kubernetes`使用`k8s.io`作为`namespace`名称。

<!--more-->

所以在使用`containerd`的`kubernetes`环境中，要使用`ctr`命令查看时，如果简单执行如下命令，是没有容器输出的:

```text-plain
[root@k8s-worker0 ~]# ctr container list
CONTAINER    IMAGE    RUNTIME
```

需要添加`namespace`参数，如:

```text-plain
[root@k8s-worker0 ~]# ctr -n k8s.io container list
0eada175936306b1d0e428a38770e505d113e82c783b86dd8b4bfa44a93f3782    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
222177e2ab84d450f7d2529757d4e7dcf5c7a2cefe046843774aa6b667f31ed4    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
2b98414edc9a525fb649f00c176ab8bfad460fe35708efe3eaf089c111eb1ee7    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
3c75a2b3d7956b7a4aa25770f7c569d1e5d817438e787782b3e9e00c80ea79c7    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
3e6f73eab4bc200c436842c7d95055224731839791957fb58b9c8baf867dee46    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
56dbc8de39cf32c9734aeb935b54d556b05ba7512d15cac2ddaa545e502d22da    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
6ee605bc01510e40341c98149f90ed6693b6557b5cd5c8836c61d01f34b2d51a    registry.k8s.io/kube-proxy:v1.29.0                                                      io.containerd.runc.v2
7228c44c6b5ad68bb8bb6aecf1fec72f121edd2dadcefe37ab4dd9869e093b32    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
7f52db24f19507da4fdd232eea23fb87e5a1ae7763e255af0f04516a7c9bb008    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
8a360bbb308aafebde4371a84000aaa5ebf300f1a4d1169dbf14ad02d3e5658d    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
a915e48150bb7d4488893261da59bed87e1a3e574e847fb966e5eeb360e2e3ed    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
b68eb4ae7d918f20689eba8bb4d109c02d2b333a3ffda3b656f30d1bf461d19c    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
b968b8f6ac6a93543ba13de31893daa1e6ddac296cacda4c8e836a2c8148d824    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
ba36dab2f3c65f2478d6a25252e7844260fdaa2af779c6f1381c02540185c29b    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
d6d4332bbfb1e8b1f179947fd1301e9646c19473c00d885e798c111291667f6c    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
e8ae98caf2f066a97197a7705159d57cc0224fa2d3ba904e814f76adb586ff2c    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
f0c578821e55910df0b8b7c3889037004db222493d85ad9ace8527d45e973be8    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
f5a33aaccdb1c968bbfcbe9428d0ae5a70b59a8618724ed27e55f85d11bb8cb4    registry.k8s.io/pause:3.9                                                               io.containerd.runc.v2
```

如果需要容器关联的`POD`信息，需要使用`crictl`:

```text-plain
[root@cluster1 ~]# crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                     ATTEMPT             POD ID              POD
0847e65e20b43       14d7d9ac51894       29 hours ago        Running             mariadb                  0                   4f2535726e06f       mysql-mariadb-0
4bb865f74a8a7       11a5d8a9f31aa       31 hours ago        Running             lb-tcp-3129              0                   5014c89cfe109       svclb-frontend-lb-06a5c6f4-r45ls
68d22a1d09ea1       11a5d8a9f31aa       31 hours ago        Running             lb-tcp-20001             0                   5014c89cfe109       svclb-frontend-lb-06a5c6f4-r45ls
6c75bf330db4a       11a5d8a9f31aa       31 hours ago        Running             lb-tcp-7902              0                   5014c89cfe109       svclb-frontend-lb-06a5c6f4-r45ls
ee41385abf1d7       11a5d8a9f31aa       31 hours ago        Running             lb-tcp-7901              0                   5014c89cfe109       svclb-frontend-lb-06a5c6f4-r45ls
7360a6ee7b30a       11a5d8a9f31aa       31 hours ago        Running             lb-tcp-443               0                   5014c89cfe109       svclb-frontend-lb-06a5c6f4-r45ls
f1dc52862daaf       3a0f7b0a13ef6       2 weeks ago         Running             registry                 0                   dcbea1d5918ca       registry-795cf4cd5-6djg2
b4b864e0a38a2       c69fa2e9cbf5f       2 weeks ago         Running             coredns                  0                   af17c0e59afc2       coredns-56f6fc8fd7-fqd87
a210c70357684       48d9cfaaf3904       2 weeks ago         Running             metrics-server           0                   db363ba6b01e3       metrics-server-5985cbc9d7-vvklc
4423e81878099       5d221316a3c61       2 weeks ago         Running             local-path-provisioner   0                   b4c91a7452716       local-path-provisioner-846b9dcb6c-48nqw
```

查询节点上所有的`POD`：

```text-plain
[root@cluster1 ~]# crictl pods
POD ID              CREATED             STATE               NAME                                      NAMESPACE           ATTEMPT             RUNTIME
5014c89cfe109       31 hours ago        Ready               svclb-frontend-lb-06a5c6f4-r45ls          kube-system         0                   (default)
af17c0e59afc2       2 weeks ago         Ready               coredns-56f6fc8fd7-fqd87                  kube-system         0                   (default)
db363ba6b01e3       2 weeks ago         Ready               metrics-server-5985cbc9d7-vvklc           kube-system         0                   (default)
b4c91a7452716       2 weeks ago         Ready               local-path-provisioner-846b9dcb6c-48nqw   kube-system         0                   (default)
```

参考:

*   [https://www.devopsschool.com/blog/what-is-difference-between-crictl-and-ctr/](https://www.devopsschool.com/blog/what-is-difference-between-crictl-and-ctr/)
*   [https://labs.iximiuz.com/courses/containerd-cli/ctr/image-management#what-is-ctr](https://labs.iximiuz.com/courses/containerd-cli/ctr/image-management#what-is-ctr)
*   [https://www.hackingnote.com/en/versus/crictl-vs-ctr/](https://www.hackingnote.com/en/versus/crictl-vs-ctr/)
*   [https://labs.iximiuz.com/courses/containerd-cli/ctr/container-management#recap-what-is-ctr](https://labs.iximiuz.com/courses/containerd-cli/ctr/container-management#recap-what-is-ctr)
*   [https://cloudyuga.guru/blogs/containerd-and-ctr/](https://cloudyuga.guru/blogs/containerd-and-ctr/)
