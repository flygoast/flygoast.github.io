---
title: 本地环境Kubernetes LoadBalancer实现
date: 2021-11-21 14:04:59
tags:
- kubernetes
- externalIP
- loadbalancer
categories: Kubernetes
---
`Kubernetes`没有给本地环境(`Bare-metal`, `On-Premise`)提供负载均衡实现，`LoadBalancer`类型的服务主要在各大公有云厂商上能够得到原生支持。在本地环境创建`LoadBalancer`类型的服务后，服务的`EXTERNAL-IP`会一直处于`<pending>`状态。这是因为在本地环境没有相应的`controller`来处理这些`LoadBalancer`服务。比如:
```bash
[root@master1 vagrant]# kubectl get svc
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.32.0.1     <none>        443/TCP        31d
whoami       LoadBalancer   10.32.0.132   <pending>     80:31620/TCP   103s
```

之前的文章[<<基于LVS DR模式的Kubernetes Service External-IP实现>>](/2021/11/14/external-ip/)介绍了手动设置`EXTERNAL-IP`的方式实现外部负载均衡。本文通过在本地环境实现一个简单的`controller`来处理`LoadBalancer`类型服务自动实现负载均衡器。架构示意如图:

{% img /images/2021-11-21/1.png %}

`LoadBalancer`是位于`Kubernetes`集群外的独立集群。可以通过`ECMP`将请求分散到不同的`LoaderBalancer`节点上，`LoadBalancer`再将请求分发到`Kubernetes`的`node`上。

<!--more-->

服务依然使用之前文章中的`whoami`服务定义，`type`修改为`LoadBalancer`:
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: whoami
  name: whoami
spec:
  ports:
  - port: 80
    name: web
    protocol: TCP
  selector:
    app: whoami
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: containous/whoami:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: web
```

创建服务:
```bash
[root@master1 vagrant]# kubectl apply -f whoami.yaml
service/whoami created
deployment.apps/whoami created
```
此时查看集群中的`service`, 可以看到`whoami`服务的`EXTERNAL-IP`处于`<pending>`状态:
```bash
[root@master1 vagrant]# kubectl get svc -o wide
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes   ClusterIP      10.32.0.1     <none>        443/TCP        31d   <none>
whoami       LoadBalancer   10.32.0.188   <pending>     80:31220/TCP   6s    app=whoami
```

接着编写自定义的`controller`来处理`LoadBalancer`, 自定义`controller`的编写可以参考之前的文章[<<Kubernetes CDR和Custom Controller>>](/2020/03/02/Kubernetes-CDR/), `main.py`源码如下:
```python
# coding: utf-8

import logging
import sys
import os
from kubernetes import client, config, watch

log = logging.getLogger(__name__)
out_hdlr = logging.StreamHandler(sys.stdout)
out_hdlr.setFormatter(logging.Formatter('%(asctime)s %(message)s'))
out_hdlr.setLevel(logging.INFO)
log.addHandler(out_hdlr)
log.setLevel(logging.INFO)


lb_ip_pools = [
    '10.240.0.210',
    '10.240.0.211',
    '10.240.0.212',
    '10.240.0.213',
    '10.240.0.214',
    '10.240.0.215'
]

node_ips = ['10.240.0.101', '10.240.0.102']


lb_services = {}
kubernetes_services = {}

def add_services(svc_manifest):
    ports = svc_manifest.spec.ports
    lb_svcs = []

    for ip in lb_ip_pools:
        if (ip not in lb_services) or (len(lb_services[ip]) == 0):
            lb_services[ip] = {}

            for port in ports:
                lb_svcs.append((port.protocol, ip, port.port))

                if port.port not in lb_services[ip]:
                    lb_services[ip][port.port] = []
                lb_services[ip][port.port].append(port.protocol)

            kubernetes_services[svc_manifest.metadata.name] = lb_svcs
            return lb_svcs

        valid_ip = True
        for port in ports:
            if port.port in lb_services[ip]:
                valid_ip = False
                break

        if valid_ip:
            for port in ports:
                lb_svcs.append((port.protocol, ip, port.port))

                if port.port not in lb_services[ip]:
                    lb_services[ip][port.port] = []
                lb_services[ip][port.port].append(port.protocol)

            kubernetes_services[svc_manifest.metadata.name] = lb_svcs
            return lb_svcs

    return None


def del_services(svc_manifest):
    lb_svcs = kubernetes_services[svc_manifest.metadata.name]
    del kubernetes_services[svc_manifest.metadata.name]

    for svc in lb_svcs:
        del lb_services[svc[1]][svc[2]]
    return lb_svcs


def del_ipvs(lb_svcs):
    for item in lb_svcs:
        if item[0] == 'TCP':
            command = "ipvsadm -D -t %s:%d" % (item[1], item[2])
            os.system(command)
        elif item[0] == 'UDP':
            command = "ipvsadm -D -u %s:%d" % (item[1], item[2])
            os.system(command)

def add_ipvs(lb_svcs):
    for item in lb_svcs:
        if item[0] == 'TCP':
            command = "ipvsadm -A -t %s:%d -s rr" % (item[1], item[2])
            os.system(command)
            for node_ip in node_ips:
                command = "ipvsadm -a -t %s:%d -r %s -g" % (item[1], item[2], node_ip)
                os.system(command)
        elif item[0] == 'UDP':
            command = "ipvsadm -A -u %s:%d -s rr" % (item[1], item[2])
            os.system(command)
            for node_ip in node_ips:
                command = "ipvsadm -a -u %s:%d -r %s -g" % (item[1], item[2], node_ip)
                os.system(command)
        else:
            log.error("invalid protocol: %s", item[0])

 def main():
    config.load_kube_config()

    v1 = client.CoreV1Api()
    w = watch.Watch()
    for item in w.stream(v1.list_service_for_all_namespaces):
        if item["type"] == "ADDED":
            svc_manifest = item['object']
            namespace = svc_manifest.metadata.namespace
            name = svc_manifest.metadata.name
            svc_type = svc_manifest.spec.type

            log.info("Service ADDED: %s %s %s" % (namespace, name, svc_type))

            if svc_type == "LoadBalancer":
                if svc_manifest.status.load_balancer.ingress == None:
                    log.info("Process load balancer service add event")
                    lb_svcs = add_services(svc_manifest)
                    if lb_svcs == None:
                        log.error("no available loadbalancer IP")
                        continue
                    add_ipvs(lb_svcs)
                    svc_manifest.status.load_balancer.ingress = [{'ip': lb_svcs[0][1]}]
                    v1.patch_namespaced_service_status(name, namespace, svc_manifest)
                    log.info("Update service status")

        elif item["type"] == "MODIFIED":
            log.info("Service MODIFIED: %s %s" % (item['object'].metadata.name, item['object'].spec.type))

        elif item["type"] == "DELETED":
            svc_manifest = item['object']
            namespace = svc_manifest.metadata.namespace
            name = svc_manifest.metadata.name
            svc_type = svc_manifest.spec.type

            log.info("Service DELETED: %s %s %s" % (namespace, name, svc_type))

            if svc_type == "LoadBalancer":
                if svc_manifest.status.load_balancer.ingress != None:
                    log.info("Process load balancer service delete event")
                    lb_svcs = del_services(svc_manifest)
                    if len(lb_svcs) != 0:
                        del_ipvs(lb_svcs)

if __name__ == '__main__':
    main()
```

我们的`controller`程序通过`Kubernetes APIServer`监控`service`对象。当检测到有`LoadBalancer`类型的`service`创建后，则分配`VIP`并在节点上创建`IPVS`服务，然后修改相应`service`的`status`。我们使用的依然是`RR`模式，将数据包分发到后端的`node`。修改目的`MAC`地址的数据包到达`node`能够正常处理是由于`kube-proxy`会根据修改后的`service`对象创建相应的`iptables`规则:
```plain
-A KUBE-SERVICES -d 10.240.0.210/32 -p tcp -m comment --comment "default/whoami:web loadbalancer IP" -m tcp --dport 80 -j KUBE-FW-225DYIB7Z2N6SCOU
```

我们的实验环境是单节点的`IPVS`, 为了简单直接在`IPVS`节点上运行我们的`controller`，因而不存在多次修改`service`状态的问题。如果在多节点的`IPVS`集群，`controller`可以通过`Puppet`,`SaltStack`或者`Ansible`等运维部署工具创建`IPVS`服务。

在`IPVS`节点上安装依赖的`kubernetes`库, 因为我们使用的`kubernetes`版本为`1.15.3`, 因而安装`11.0.0`版本, 具体版本依赖参考[官网说明](https://github.com/kubernetes-client/python)
```bash
pip3 install kubernetes==11.0.0
```

运行我们的`controller`:
```bash
[root@lb1 ipvslb]# python3 main.py
2021-11-21 04:18:25,018 Service ADDED: default kubernetes ClusterIP
2021-11-21 04:18:25,019 Service ADDED: default whoami LoadBalancer
2021-11-21 04:18:25,019 Process load balancer service add event
2021-11-21 04:18:25,064 Update service status
2021-11-21 04:18:25,066 Service MODIFIED: whoami LoadBalancer


```

此时再去查看`service`:
```bash
[root@master1 vagrant]# kubectl get svc
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)        AGE
kubernetes   ClusterIP      10.32.0.1     <none>         443/TCP        31d
whoami       LoadBalancer   10.32.0.132   10.240.0.210   80:31620/TCP   2m40s
```

可以看到`whoami`服务的`EXTERNAL-IP`获取到了分配的`VIP: 10.240.0.210`。查看`IPVS`服务, 也可以看到对应的服务已经建立:

```bash
[root@lb1 ipvslb]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.240.0.210:80 rr
  -> 10.240.0.101:80              Route   1      0          0
  -> 10.240.0.102:80              Route   1      0          0
```

此时去访问`VIP`, 访问成功:
```bash
[root@master1 vagrant]# curl http://10.240.0.210
Hostname: whoami-6756777fd4-4lpvl
IP: 127.0.0.1
IP: ::1
IP: 10.230.64.2
IP: fe80::ec48:94ff:fea0:db31
RemoteAddr: 10.230.10.0:45298
GET / HTTP/1.1
Host: 10.240.0.210
User-Agent: curl/7.29.0
Accept: */*

[root@master1 vagrant]# curl http://10.240.0.210
Hostname: whoami-6756777fd4-qzm6r
IP: 127.0.0.1
IP: ::1
IP: 10.230.10.19
IP: fe80::60b7:2aff:feab:68c2
RemoteAddr: 10.230.64.0:45304
GET / HTTP/1.1
Host: 10.240.0.210
User-Agent: curl/7.29.0
Accept: */*

```

接着删除`whoami`服务:
```bash
[root@master1 vagrant]# kubectl delete svc/whoami
service "whoami" deleted
```

从`controller`输出可以看到事件被正确处理:
```bash
2021-11-21 04:27:34,046 Service DELETED: default whoami LoadBalancer
2021-11-21 04:27:34,047 Process load balancer service delete event
```

查看`IPVS`服务, 也被正确删除了:
```bash
[root@lb1 ipvslb]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
```


在我们的示例中，`VIP`池中的`IP`需要提前在`IPVS`节点配置好，比如:
```bash
[root@lb1 ipvslb]# ip a  show eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:48:90:6c brd ff:ff:ff:ff:ff:ff
    inet 10.240.0.6/24 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.201/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.210/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.211/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.212/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.213/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.214/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.240.0.215/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe48:906c/64 scope link
       valid_lft forever preferred_lft forever
```

`node`的`IP`也是在代码里提前配置的，正常应该从`Kubernetess APIServer`获取。在示例里简单处理了。

[metallb](https://metallb.universe.tf/)是当前使用范围较广的本地环境`LoadBalancer`实现，它本身没有使用外部独立的负载均衡集群，是在`node`节点上实现的负载均衡功能。很多`Kubernetes`产品已经集成了它，后续有时间可以分析一下它的源码实现。

参考:
* https://developer-docs.citrix.com/projects/citrix-k8s-ingress-controller/en/latest/network/type_loadbalancer/
* https://docs.openshift.com/container-platform/4.9/networking/metallb/about-metallb.html
* https://kubernetes.github.io/ingress-nginx/deploy/baremetal/
* https://docs.k0sproject.io/main/examples/metallb-loadbalancer/
