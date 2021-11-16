---
title: 基于LVS DR模式的Kubernetes Service External-IP实现
date: 2021-11-14 22:34:27
tags:
- kubernetes
- externalIP
categories: Kubernetes
---
之前的文章[<<Kubernetes Service网络通信路径>>](/2021/11/07/kubernetes-clusterip/)介绍了`kubernetes`的几种`Service`。如果要暴露服务给`kubernetes`集群外使用，可以选择`NodePort`和`LoadBalancer`。但`LoadBalancer`现在主要在各大公有云厂商上能够原生支持。而使用`NodePort`暴露服务，将使用一个非常大的端口，无法使用原始的端口号来暴露服务，比如`mysql`的`3306`端口。

`Service`的[官方文档](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips)中介绍了一种辅助方式, 叫`External-IP`, 可以在`worker`节点上会通过该`IP`来暴露服务，而且可以使用在任意类型的`service`上。集群外的用户就可以通过该`IP`来访问服务。但如果这个`IP`只存在于一个`worker`节点上，那么就不具备高可用的能力了，我们需要在多个`worker`节点上配置这个`VIP:Virtual IP`。我们可以使用`LVS`(也叫做`IPVS`)的`DR(Director Routing)`模式作为外部负载均衡器将流量分发到多个`worker`节点上，同时保持数据包的目的地址为该`VIP`。

`DR`模式只会修改数据包的目的`MAC`地址为后端`RealServer`的`MAC`地址，因而要求负载均衡器`Director`和`RealServer`在同一个二层网络，而且响应包不会经过`Director`。

<!--more-->

下面我们来实验如何使用`LVS`的`DR`模式实现`service`负载均衡。

我们在之前的实验集群中创建一个类型为`ClusterIP`(默认类型)的`service`, 指定一个外部`IP`:
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
  externalIPs:
  - 10.240.0.201
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
创建服务:
```bash
kubectl apply -f whoami.yaml
```

查看服务，可以看到`whoami`的`EXTERNAL-IP`为`10.240.0.201`:
```bash
[root@master1 ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP    PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.32.0.1    <none>         443/TCP   24d   <none>
whoami       ClusterIP   10.32.0.60   10.240.0.201   80/TCP    30m   app=whoami
```

在`worker`节点上检查`iptables`规则，可以看到在`KUBE-SERVICES`链中添加了`EXTERNAL-IP`相关的规则:
```plain
-A KUBE-SERVICES ! -s 10.230.0.0/16 -d 10.32.0.60/32 -p tcp -m comment --comment "default/whoami:web cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.32.0.60/32 -p tcp -m comment --comment "default/whoami:web cluster IP" -m tcp --dport 80 -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES -d 10.240.0.201/32 -p tcp -m comment --comment "default/whoami:web external IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.240.0.201/32 -p tcp -m comment --comment "default/whoami:web external IP" -m tcp --dport 80 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES -d 10.240.0.201/32 -p tcp -m comment --comment "default/whoami:web external IP" -m tcp --dport 80 -m addrtype --dst-type LOCAL -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES ! -s 10.230.0.0/16 -d 10.32.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.32.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```
当数据包目的地址为`10.240.0.201:80`时，跳转到`KUBE-SVC-*`链，从而分发到相应的`pod`中。

我们在节点上添加上这个`VIP`:
```bash
[root@node1 ~]# ip addr add 10.240.0.201/32 dev lo
[root@node1 ~]# ip addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.240.0.201/32 scope global lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```
因为这个`VIP`需要在多个`worker`节点上存在，因而把它配置在`lo`上，并抑制相应网卡上对该`VIP`的`ARP`响应:
```bash
sysctl -w net.ipv4.conf.eth1.arp_ignore = 1
sysctl -w net.ipv4.conf.eth1.arp_announce = 2
```

在节点上尝试访问`VIP`, 可以成功访问:
```bash
[root@node1 ~]# curl http://10.240.0.201
Hostname: whoami-5df4df6ff5-kbv68
IP: 127.0.0.1
IP: ::1
IP: 10.230.95.10
IP: fe80::d43a:9eff:fe3e:4425
RemoteAddr: 10.230.74.0:60086
GET / HTTP/1.1
Host: 10.240.0.201
User-Agent: curl/7.29.0
Accept: */*

[root@node1 ~]# curl http://10.240.0.201
Hostname: whoami-5df4df6ff5-n6jmj
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.25
IP: fe80::9889:dff:fedf:f376
RemoteAddr: 10.230.74.1:60088
GET / HTTP/1.1
Host: 10.240.0.201
User-Agent: curl/7.29.0
Accept: */*

[root@node1 ~]# curl http://10.240.0.201
Hostname: whoami-5df4df6ff5-2h6qf
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.24
IP: fe80::2493:9aff:fe7b:5dbd
RemoteAddr: 10.230.74.1:60090
GET / HTTP/1.1
Host: 10.240.0.201
User-Agent: curl/7.29.0
Accept: */*

```

接着我们在`worker`节点所在二层网络再启动一台虚拟机作为`LVS`的`Director`。在该机器上给与`worker`节点二层互通的网卡添加`VIP`:
```bash
ip addr add 10.240.0.201/32 dev eth1
```

使用`ipvsadm`创建负载均衡服务, 并使用`DR`模式添加两个`worker`节点做为后端的`RealServer`:
```bash
ipvsadm -A -t 10.240.0.201:80 -s rr
ipvsadm -a -t 10.240.0.201:80 -r 10.240.0.101 -g
ipvsadm -a -t 10.240.0.201:80 -r 10.240.0.102 -g
```

查看负载均衡服务:
```bash
[root@lb1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.240.0.201:80 rr
  -> 10.240.0.101:80              Route   1      0          0
  -> 10.240.0.102:80              Route   1      0          0
```

环境配置完成。我们找一台客户端访问`VIP:10.240.0.201`, 同时在`Director`机器上抓包，可以看到:
```bash
[root@lb1 ~]# tcpdump -ieth1 -nn -e tcp port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
11:50:01.024615 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: 10.240.0.10.38482 > 10.240.0.201.80: Flags [S], seq 1959573689, win 29200, options [mss 1460,sackOK,TS val 304318064 ecr 0,nop,wscale 6], length 0
11:50:01.024640 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 74: 10.240.0.10.38482 > 10.240.0.201.80: Flags [S], seq 1959573689, win 29200, options [mss 1460,sackOK,TS val 304318064 ecr 0,nop,wscale 6], length 0
11:50:01.026358 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 3346334225, win 457, options [nop,nop,TS val 304318066 ecr 304104626], length 0
11:50:01.026406 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 1, win 457, options [nop,nop,TS val 304318066 ecr 304104626], length 0
11:50:01.027197 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 142: 10.240.0.10.38482 > 10.240.0.201.80: Flags [P.], seq 0:76, ack 1, win 457, options [nop,nop,TS val 304318067 ecr 304104626], length 76: HTTP: GET / HTTP/1.1
11:50:01.027210 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 142: 10.240.0.10.38482 > 10.240.0.201.80: Flags [P.], seq 0:76, ack 1, win 457, options [nop,nop,TS val 304318067 ecr 304104626], length 76: HTTP: GET / HTTP/1.1
11:50:01.032443 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 327, win 473, options [nop,nop,TS val 304318070 ecr 304104630], length 0
11:50:01.032468 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 327, win 473, options [nop,nop,TS val 304318070 ecr 304104630], length 0
11:50:01.036452 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [F.], seq 76, ack 327, win 473, options [nop,nop,TS val 304318072 ecr 304104630], length 0
11:50:01.037159 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [F.], seq 76, ack 327, win 473, options [nop,nop,TS val 304318072 ecr 304104630], length 0
11:50:01.047556 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 328, win 473, options [nop,nop,TS val 304318087 ecr 304104647], length 0
11:50:01.047583 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 328, win 473, options [nop,nop,TS val 304318087 ecr 304104647], length 0
```

数据包的目的`MAC`地址被修改为`node2`上`eth1`的`MAC`地址, 而且响应包并不经过`Director`:
```bash
[root@node2 ~]# ip link show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:23:1b:95 brd ff:ff:ff:ff:ff:ff
```


根据之前网上的[这篇文章](https://jishu.io/kubernetes/ipvs-loadbalancer-for-kubernetes/), `worker`节点可以不设置`VIP`，因为`VIP`并不需要由用户态程序来接收流量，而是直接由`iptables`来进行数据包转换。

在大多数场景下这是正确的。但是如果需要直接从`worker`节点上通过`VIP`访问该服务时就需要在`worker`节点上配置`VIP`了。数据包在从`worker`节点发送出去时，会经由`nat:OUTPUT`和`nat:POSTROUTING`来处理。`iptables`的`NAT`实现是基于`conntrack`来实现的，发出时系统会建立`conntrack`条目。`iptables`的`nat:PREROUTING`和`nat:POSTROUTING`的处理都会调用`nf_nat_ipv4_fn`函数。当数据包由`LVS Director`把数据包返回到自身这台`RealServer`时, `nat:PREROUTING`阶段调用`nf_nat_ipv4_fn`:
```c
    switch (ctinfo) {
    case IP_CT_RELATED:
    case IP_CT_RELATED_REPLY:
        if (ip_hdr(skb)->protocol == IPPROTO_ICMP) {
            if (!nf_nat_icmp_reply_translation(skb, ct, ctinfo,
                               ops->hooknum))
                return NF_DROP;
            else
                return NF_ACCEPT;
        }
        /* Fall thru... (Only ICMPs can be IP_CT_IS_REPLY) */
    case IP_CT_NEW:
        /* Seen it before?  This can happen for loopback, retrans,
         * or local packets.
         */
        if (!nf_nat_initialized(ct, maniptype)) {
            unsigned int ret;

            ret = do_chain(ops, skb, state, ct);
            if (ret != NF_ACCEPT)
                return ret;

            if (nf_nat_initialized(ct, HOOK2MANIP(ops->hooknum)))
                break;

            ret = nf_nat_alloc_null_binding(ct, ops->hooknum);
            if (ret != NF_ACCEPT)
                return ret;
        } else {
            pr_debug("Already setup manip %s for ct %p\n",
                 maniptype == NF_NAT_MANIP_SRC ? "SRC" : "DST",
                 ct);
            if (nf_nat_oif_changed(ops->hooknum, ctinfo, nat,
                           state->out))
                goto oif_changed;
        }
        break;

    default:
        /* ESTABLISHED */
        NF_CT_ASSERT(ctinfo == IP_CT_ESTABLISHED ||
                 ctinfo == IP_CT_ESTABLISHED_REPLY);
        if (nf_nat_oif_changed(ops->hooknum, ctinfo, nat, state->out))
            goto oif_changed;
    }
```

这时`nf_nat_initialized`会返回`0`, 因而跳过`do_chain`的调用，也就不再会执行`nat:PREROUTING`所设置的链和规则，放行通过进入到路由决策阶段。但由于数据包的源`IP`是本机地址，默认情况下Linux路由实现不允许非`loopback`设备之外的设备所进入的数据包源地址为本机地址, 因而该数据包会被丢弃。

但内核提供了参数`accept_local`可以允许这种包通过:
```plain
accept_local - BOOLEAN
    Accept packets with local source addresses. In combination
    with suitable routing, this can be used to direct packets
    between two local interfaces over the wire and have them
    accepted properly.

    rp_filter must be set to a non-zero value in order for
    accept_local to have an effect.


rp_filter - INTEGER
    0 - No source validation.
    1 - Strict mode as defined in RFC3704 Strict Reverse Path
        Each incoming packet is tested against the FIB and if the interface
        is not the best reverse path the packet check will fail.
        By default failed packets are discarded.
    2 - Loose mode as defined in RFC3704 Loose Reverse Path
        Each incoming packet's source address is also tested against the FIB
        and if the source address is not reachable via any interface
        the packet check will fail.

    Current recommended practice in RFC3704 is to enable strict mode
    to prevent IP spoofing from DDos attacks. If using asymmetric routing
    or other complicated routing, then loose mode is recommended.

    The max value from conf/{all,interface}/rp_filter is used
    when doing source validation on the {interface}.

    Default value is 0. Note that some distributions enable it
    in startup scripts.
```

修改相应参数放行数据包:
```bash
sysctl -w net.ipv4.conf.eth1.rp_filter=1
sysctl -w net.ipv4.conf.eth1.accept_local=1
```

再次从`worker`节点访问`VIP`, 同时开启`tcpdump`抓包分析:
```bash
[root@node1 ~]# curl http://10.240.0.201
curl: (7) Failed connect to 10.240.0.201:80; No route to host
```

可以看到返回路由错误，而从抓包结果看，我们放行数据包后，根据目的地址，数据包会再被发送出去，从而形成环路。直到`IP`包的`ttl`减为`0`返回了路由错误。

```bash
[root@node1 ~]# tcpdump -ieth1 tcp port 80 -nn -e -v
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
02:57:18.801428 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 64, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0x173c (incorrect -> 0xc00b), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:18.801803 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 64, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:18.814275 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 63, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:18.814579 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 63, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0

...

02:57:19.054672 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 2, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:19.054982 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 2, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:19.057681 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 1, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:19.057978 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 1, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
```

本文只是简单实验可行性。如果用于生产环境，需要额外的方案考虑，比如:
* `LVS`本身可以配合`keepalived`使用主备模式保证`Director`的HA
* 使用`OSPF`的`ECMP`来配置多主的`Director`集群(可以参考之前的文章[<<基于Cumulus VX实验ECMP+OSPF负载均衡>>](/2020/05/05/emcp-ospf-cumulus/))
* 省略`LVS`的`Director`层，直接使用`OSPF`的`ECMP`将流量分发到`worker`节点的`VIP`

参考:
* https://medium.com/swlh/kubernetes-external-ip-service-type-5e5e9ad62fcd
* http://www.361way.com/lvs-dr-theory/5195.html
* https://jishu.io/kubernetes/ipvs-loadbalancer-for-kubernetes/
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
