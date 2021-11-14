---
title: Kubernetes Service网络通信分析
date: 2021-11-07 23:26:39
tags:
- Kubernetes
- Iptables
- Kube-proxy
categories: Kubernetes
---
之前的文章[<<Kubernetes网络和CNI>>](/2020/07/25/kubernetes-cni/)和[<<Kubernetes flannel网络分析>>](/2021/11/03/flannel/)介绍了Kubernetes集群的`pod`网络的通信过程。

`pod`本质上非固定的，经常发生变化，而`pod IP`在`pod`销毁和创建的时候会发生变更，因而不能直接对外提供服务。`Kubernetes`通过`service`资源来对外提供服务，`service`的`IP`是固定的，它自动绑定一组`pod`并根据不同实现将流量转发到这些`pod`中, 并在流量转发的过程中实现负载均衡(load balance)。

在`Kubernetes`的`node`节点上的主要组件有`kube-proxy`和`kubelet`, `kubelet`会调用相关的`CNI`实现完成`POD`网络的通信。而`kube-proxy`则负责上述的`service`与`POD`之间的流量转发。实际上，在之前文章的实验环境里，即使把`node`节点上的`kube-proxy`组件都停止，也不会影响`pod`网络通信。

`service`本质就是将一组`pod`通过固定`IP`暴露给使用者，可以由`ip:port:protocol`来标识。

`service`主要以下几种类型:

* `ClusterIP`: 用`Kubernetes`集群内部`IP`暴露服务，也就是说只有在`Kubernetes`集群内才可以访问这个`service`。这是默认的`service`类型。`ClusterIP`的范围是在`kube-apiserver`启动时通过`-service-cluster-ip-range`参数指定的。这些`IP`只能在`kubernetes`集群内进行访问。`service`的相关信息是在`yaml`文件中定义的，最终暴露的信息可表示为:
```plain
spec.clusterIp:spec.ports[*].port:spec.ports[*].protocol
```

* `NodePort`: 在`Kubernetes`集群的所有`node`节点上使用相同的固定`port`来暴露服务。这种类型会自动创建`ClusterIP`类型的服务，`NodePort`的`service`会将流量转发到`ClusterIP`类型的服务。服务的使用者可以使用`NodeIP:NodePort`来访问该服务。这种类型服务暴露的信息可以表示为:
```
<NodeIP>:spec.ports[*].nodePort:spec.ports[*].protocol
spec.clusterIp:spec.ports[*].port:spec.ports[*].protocol
```

* `LoadBalancer`: 是通过`kubernetes`集群外部设施所提供的`IP`来暴露服务。`NodePort`和`ClusterIP`类型的服务会被自动创建。不同的`LoadBalancer`负责实现外部`IP:port`与`NodePort`服务的映射。这种类型暴露的信息可以表示为:
```
spec.loadBalancerIp:spec.ports[*].port:spec.ports[*].protocol
<NodeIP>:spec.ports[*].nodePort:spec.ports[*].protocol
spec.clusterIp:spec.ports[*].port:spec.ports[*].protocol
```

<!--more-->

`Kubernetes`集群默认会存在一个`ClusterIP`类型的服务`kubernetes`，它可以让集群内部的`pod`去访问`kube-apiserver`，如:
```bash
[root@master1 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP   15d
```

接下来我们基于`containous/whoami`镜像创建一个`ClusterIP`类型的服务`whoami`:
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
  type: ClusterIP
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
        image: containous/whoami
        ports:
        - containerPort: 80
          name: web
```

创建`service`:
```bash
[root@master1 ~]# kubectl apply -f whoami.yaml
service/whoami created
deployment.apps/whoami created
```

查看`service`资源:
```bash
[root@master1 ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes   ClusterIP   10.32.0.1     <none>        443/TCP   16d    <none>
whoami       ClusterIP   10.32.0.235   <none>        80/TCP    3m7s   app=whoami
```

查看`service`对应的`endpoint`:
```bash
[root@master1 ~]# kubectl get ep
NAME         ENDPOINTS                                            AGE
kubernetes   10.240.0.10:6443,10.240.0.11:6443,10.240.0.12:6443   16d
whoami       10.230.74.7:80,10.230.74.8:80,10.230.95.7:80         3h23m
```

创建一个客户端`curl`的`pod`:
```bash
[root@master1 ~]# kubectl run curl1 --image=radial/busyboxplus:curl --command -- sleep 3600
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/curl1 created
```
此时`pod`如下, 一个`curl`客户端，3个`whoami`的WEB服务器:
```bash
[root@master1 ~]# kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
curl1-54dbd6b8cb-vm5wc    1/1     Running   0          8m1s   10.230.74.19   node1   <none>           <none>
whoami-5df4df6ff5-2frx6   1/1     Running   0          3h     10.230.74.7    node1   <none>           <none>
whoami-5df4df6ff5-84tpp   1/1     Running   0          3h     10.230.95.7    node2   <none>           <none>
whoami-5df4df6ff5-rh4zq   1/1     Running   0          3h     10.230.74.8    node1   <none>           <none>
```

从客户端`pod`访问`service`, 可以看到请求被转发到不同的后端`pod`:
```bash
[root@master1 ~]# kubectl exec -it curl1-54dbd6b8cb-vm5wc -- curl http://10.32.0.235
Hostname: whoami-5df4df6ff5-rh4zq
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.8
IP: fe80::589f:13ff:fef7:bc05
RemoteAddr: 10.230.74.19:36694
GET / HTTP/1.1
Host: 10.32.0.235
User-Agent: curl/7.35.0
Accept: */*

[root@master1 ~]# kubectl exec -it curl1-54dbd6b8cb-vm5wc -- curl http://10.32.0.235
Hostname: whoami-5df4df6ff5-84tpp
IP: 127.0.0.1
IP: ::1
IP: 10.230.95.7
IP: fe80::5433:63ff:fe98:6d1e
RemoteAddr: 10.230.74.0:36698
GET / HTTP/1.1
Host: 10.32.0.235
User-Agent: curl/7.35.0
Accept: */*

[root@master1 ~]# kubectl exec -it curl1-54dbd6b8cb-vm5wc -- curl http://10.32.0.235
Hostname: whoami-5df4df6ff5-2frx6
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.7
IP: fe80::20c8:faff:fe77:7a40
RemoteAddr: 10.230.74.19:36706
GET / HTTP/1.1
Host: 10.32.0.235
User-Agent: curl/7.35.0
Accept: */*
```

`kube-proxy`服务会监听`kube-apiserver`中的`service`和`endpoint`的变化来配置`service`和`endpoint`的对应关系。当前具体的转发实现方案有`userspace`, `iptables`, `ipvs`几种，默认实现为`iptables`。本文来分析`iptables`实现方案下，`POD`访问`ClusterIP`的数据包路径。数据包整体通过`iptables`链的过程，可以参考之前的文章[<<IPTABLES机制分析>>](/2016/12/11/iptables/)。

要注意的是，`kube-proxy`的`iptables`规则都是添加在`init_net`的，所以数据包从`pod`一端的`veth peer`发出，然后从宿主机这端`veth peer`进入`init_net`时，才会由`nat`表的`PREROUTING`链处理, 跳转至`KUBE-SERVICES`链:
```bash
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

`KUBE-SERVICES`链规则如下:
```bash
-A KUBE-SERVICES ! -s 10.230.0.0/16 -d 10.32.0.235/32 -p tcp -m comment --comment "default/whoami:web cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.32.0.235/32 -p tcp -m comment --comment "default/whoami:web cluster IP" -m tcp --dport 80 -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES ! -s 10.230.0.0/16 -d 10.32.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.32.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

`kubernetes`和`whoami`两个`service`, 每个`service`有两条`iptables`规则:
第一条表示，`! -s 10.230.0.0/16`表示数据包来自`kubernetes`集群外`IP`。如果是`kubernetes`集群外访问`service`，跳转至`KUBE-MARK-MASQ`。
第二条规则表示，集群内`IP`访问该`service`则跳转至`KUBE-SVC-225DYIB7Z2N6SCOU`处理。

最后一条`KUBE-NODEPORTS`用于匹配`NodePort`类型的`service`。

`KUBE-SVC-225DYIB7Z2N6SCOU`的规则如下:
```bash
-A KUBE-SVC-225DYIB7Z2N6SCOU -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-6ALQJ7ITJYIQGLX3
-A KUBE-SVC-225DYIB7Z2N6SCOU -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-GEJ3N4FTSFGJDJNF
-A KUBE-SVC-225DYIB7Z2N6SCOU -j KUBE-SEP-VVIWAZ76Z4TWVPK7
```
`KUBE-SVC-225DYIB7Z2N6SCOU`的规则，使用`statistic`模块的`random`模式来实现负载均衡，跳转到不同的`endpoint`。每个`endpoint`有一条自定义链，如`KUBE-SEP-6ALQJ7ITJYIQGLX3`:
```bash
-A KUBE-SEP-6ALQJ7ITJYIQGLX3 -s 10.230.74.7/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-6ALQJ7ITJYIQGLX3 -p tcp -m tcp -j DNAT --to-destination 10.230.74.7:80
```
第一条规则表示数据包是由`service`的`endpoint`自身去访问`ClusterIP`而访问到自身, 这种情况跳转至`KUBE-MARK-MASQ`处理。
第二条规则表示源不是自身`pod IP`的数据包，这种情况下将数据包目标地址的`ClusterIP:Port`修改为对应`pod`的地址和端口。

之后数据包进行路由决策，判断是由`flannel.1`发送到其他节点，还是由`cni0`进行二层转发。

进入`filter`表的`FORWARD`链进行处理如下:
```bash
-A FORWARD -m comment --comment "kubernetes forwarding rules" -j KUBE-FORWARD
-A FORWARD -m conntrack --ctstate NEW -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

而`KUBE-FORWARD`的规则如下:
```bash
-A KUBE-FORWARD -m conntrack --ctstate INVALID -j DROP
-A KUBE-FORWARD -m comment --comment "kubernetes forwarding rules" -m mark --mark 0x4000/0x4000 -j ACCEPT
-A KUBE-FORWARD -s 10.230.0.0/16 -m comment --comment "kubernetes forwarding conntrack pod source rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A KUBE-FORWARD -d 10.230.0.0/16 -m comment --comment "kubernetes forwarding conntrack pod destination rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

`FORWARD`阶段主要是由`conntrack`模块实现连接跟踪。连接状态为`NEW`的数据包跳转到`KUBE-SERVICE`链进行处理。需要注意的是，`iptables`中不同表中的同名链是不相同的链。这里`filter`表的`KUBE-SERVICES`链为空。

数据包路由决策结束之后进入`nat`的`POSTROUTING`处理:
```bash
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -s 10.230.74.7/32 -m comment --comment "name: \"cbr0\" id: \"d2ec32c253b68ec5b3c5327c51c62637e8a54f0f96046f4c7820fd671c0c99c8\"" -j CNI-b227064a304757ab8c96d7e0
-A POSTROUTING -s 10.230.74.8/32 -m comment --comment "name: \"cbr0\" id: \"6ddf1a0d25326e975fb4a36f441ff86cb52977de18fbc10d5e153e1ab89ddd6f\"" -j CNI-e4c4a4418638b487abe587d5
-A POSTROUTING -s 10.230.74.19/32 -m comment --comment "name: \"cbr0\" id: \"f75b50453a13f79d2fe56f288b04b248a3194f76ceb56f3a0e55b083cdedc006\"" -j CNI-3f43e40b0646cf4ef37761a7
```

`KUBE-POSTROUTING`的规则如下:
```bash
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

客户端`pod`发出的数据包匹配到`nat:POSTROUTING`的第4条规则，跳到转`CNI-3f43e40b0646cf4ef37761a7`处理:
```bash
-A CNI-3f43e40b0646cf4ef37761a7 -d 10.230.74.0/24 -m comment --comment "name: \"cbr0\" id: \"f75b50453a13f79d2fe56f288b04b248a3194f76ceb56f3a0e55b083cdedc006\"" -j ACCEPT
-A CNI-3f43e40b0646cf4ef37761a7 ! -d 224.0.0.0/4 -m comment --comment "name: \"cbr0\" id: \"f75b50453a13f79d2fe56f288b04b248a3194f76ceb56f3a0e55b083cdedc006\"" -j MASQUERADE
```
匹配到第一条规则，数据包被接受，转发到目标`pod`。

整个匹配过程如图:
{% img /images/2021-11-07/1.png %}

`pod`的响应数据包进入`init_net`之后同样首先匹配`nat:PREROUTING`，匹配不到任何规则，进入`filter`表的`FORWARD`链处理, 匹配到`conntrack`规则，此时根据`conntrack`的信息，自动完成`SNAT`过程，将响应数据包的`源IP`修改为`ClusterIP: 10.230.0.235`。

接着进入`nat`的`POSTROUTING`, 匹配不到任何规则，转发回客户端`pod`，完成一来一回的通信过程。

需要注意的是，这种情况下,`node`节点的参数`net.bridge.bridge-nf-call-iptables`需要设置为`1`。否则，当选择到同一台主机的后端`pod`时，在`KUBE-SEP-*`链中完成`DNAT`之后，目的地址和源地址在同一个二层网络内，直接由网桥转发到相应的网卡上，并不会调用到`iptables`的`FORWARD`链。这样，响应数据包回到网桥时，因为不会调用`iptables`的`FORWARD`链，不会完成对应的`SNAT`操作。因而客户端`pod`会收到源`IP`为后端`pod IP`的数据包。对于`TCP`连接，因为收到的`SYN+ACK`包不匹配，因而会发送一个`RESET`包，因而`TCP`连接无法建立。

这种场景下，在客户端`pod`上抓包，可以看到:
```bash
[root@node1 ~]# tcpdump -i vethb4eaf1e5 -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vethb4eaf1e5, link-type EN10MB (Ethernet), capture size 262144 bytes


13:20:38.386567 IP 10.230.74.19.36450 > 10.32.0.235.80: Flags [S], seq 1431048683, win 28200, options [mss 1410,sackOK,TS val 52241475 ecr 0,nop,wscale 7], length 0
13:20:38.386609 IP 10.230.74.19.36450 > 10.230.74.8.80: Flags [S], seq 1431048683, win 28200, options [mss 1410,sackOK,TS val 52241475 ecr 0,nop,wscale 7], length 0
13:20:38.386631 IP 10.230.74.8.80 > 10.230.74.19.36450: Flags [S.], seq 1948187370, ack 1431048684, win 27960, options [mss 1410,sackOK,TS val 52241475 ecr 52241475,nop,wscale 7], length 0
13:20:38.386639 IP 10.230.74.19.36450 > 10.230.74.8.80: Flags [R], seq 1431048684, win 0, length 0
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```
收到的`SYN+ACK`包的源`IP`并不是`10.32.0.235`, 而是后端`pod`的`10.230.74.8`，因而返回一个`RESET`数据包。

如果`KUBE-SVC-*`链里匹配到的`endpoint`不在该`node`, 路由决策完会从`flannel.1`发送。数据包通过`VXLAN`到达`node2`之后，数据包地址已经过`DNAT`，目标地址为`pod IP`，就是正常的`POD`网络通信过程，经由`nat:PREROUTING`，`filter:FORWARD`和`nat:POSTROUTING`处理，匹配不到任何规则，转发到目标`POD`。

响应包在`node2`上是正常的`POD`网络通信过程，通过`VXLAN`回到`node1`。然后进入`nat:PREROUTING`链处理, 匹配不到任何规则，进入`filter:FORWARD`链处理, 匹配到`conntrack`规则，根据`conntrack`的信息，自动完成`SNAT`过程，数据包的`源IP`修改回`10.230.0.235`。接着进入`nat:POSTROUTING`处理, 匹配不到任何规则，转发回客户端`pod`，完成一来一回的数据包过程。

`POD`访问`ClusterIP`的基础过程就是这样。

我们把上边的`service`修改为`NodePort`类型。
```bash
[root@master1 ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes   ClusterIP   10.32.0.1     <none>        443/TCP        18d   <none>
whoami       NodePort    10.32.0.235   <none>        80:31554/TCP   33h   app=whoami
```
查看`service`信息，分配的`port`为`31554`,

访问`NodeIP:NodePort`的数据包进入`node1`之后，先由`nat:PREROUTING`链处理, 匹配到`KUBE-SERVICES`中的规则，进而由`KUBE-NODEPORTS`进行处理:
```
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

配置`NodePort`类型`service`后，`iptables`增加了两条规则:
```plain
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/whoami:web" -m tcp --dport 31554 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/whoami:web" -m tcp --dport 31554 -j KUBE-SVC-225DYIB7Z2N6SCOU
```

在`KUBE-MARK-MASQ`链中设置`0x4000/0x4000`的`mark`值:
```plain
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```

然后跳转到`KUBE-SVC-225DYIB7Z2N6SCOU`进行处理，之后的过程和`ClusterIP`的通信过程一致，直到`nat:POSTROUTING`链的处理:
```plain
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```
因为数据包在`PREROUTING`阶段已经设置`mark`, 匹配到这条规则完成`SNAT`过程，因而在这种情况下，后端`pod`看到的源`IP`为所在`node`上`MASQUERADE`选择的`IP`。如果选定的后端`pod`位于本`node`上，则出口设备为网桥`cni0`，因而后端`pod`上看到的源`IP`为`cni0`的地址。如果选定的后端位于其他`node`, 需要通过`VXLAN`转发到其他节点，则出口设备为`flannel.1`, 因而后端`pod`上看到的源`IP`为该`node`上`flannel.1`的`IP`, 比如:

```bash
[root@master1 ~]# kubectl exec -it curl1-54dbd6b8cb-vm5wc -- curl http://10.240.0.101:31554
Hostname: whoami-5df4df6ff5-2frx6
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.7
IP: fe80::20c8:faff:fe77:7a40
RemoteAddr: 10.230.74.1:37722
GET / HTTP/1.1
Host: 10.240.0.101:31554
User-Agent: curl/7.35.0
Accept: */*

[root@master1 ~]# kubectl exec -it curl1-54dbd6b8cb-vm5wc -- curl http://10.240.0.101:31554
Hostname: whoami-5df4df6ff5-84tpp
IP: 127.0.0.1
IP: ::1
IP: 10.230.95.7
IP: fe80::5433:63ff:fe98:6d1e
RemoteAddr: 10.230.74.0:37726
GET / HTTP/1.1
Host: 10.240.0.101:31554
User-Agent: curl/7.35.0
Accept: */*
```

总结来说，`kube-proxy`的`iptables`模式主要用到这些自定义链:
* `nat:KUBE-SERVICES`: 匹配数据包目标地址，跳转到相应的`KUBE-SVC-*`
* `nat:KUBE-SVC-*`: 作为负载均衡设备，分发流量到不同的`KUBE-SEP-*`
* `nat:KUBE-SEP-*`: 代表一个后端`pod`, 也叫`service end-point`，完成`DNAT`过程
* `nat:KUBE-MARK-MASQ`: 给来自集群外的数据包设置`mark`, 这些数据包在`POSTROUTING`阶段需要完成`SNAT`操作
* `nat:KUBE-NODEPORTS`: 当配置`NodePort`或者`LoadBalancer`类型的`service`时，跳转到`KUBE-MARK-MASQ`设置`mark`和跳转到相应的`KUBE-SVC-*`


参考:
* https://arthurchiao.art/blog/cracking-k8s-node-proxy/#1-background-knowledge
* https://www.cloudsavvyit.com/11261/kubernetes-clusterip-nodeport-or-ingress-when-to-use-each/
* https://kubernetes.io/docs/concepts/services-networking/service/
* https://stackoverflow.com/questions/41509439/whats-the-difference-between-clusterip-nodeport-and-loadbalancer-service-types
* https://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy/
* https://faun.pub/kubernetes-without-kube-proxy-1c5d25786e18
* https://bugzilla.redhat.com/show_bug.cgi?id=512206
* https://blog.csdn.net/flxzlxb/article/details/112849468
* https://www.codeleading.com/article/46422763270/
* https://serenafeng.github.io/2020/03/26/kube-proxy-in-iptables-mode/
