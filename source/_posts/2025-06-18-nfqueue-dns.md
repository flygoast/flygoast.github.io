---
title: NFQUEUE机制导致DNS请求超时
date: 2025-06-18 17:18:56
tags:
- Kernel
- Nfqueue
categories: Kernel
---
之前的文章[<<NFQUEUE机制导致DNS请求5秒超时分析>>](/2021/12/29/nfqueue-dns/)分析过在`3.10.0-1015.el7`版本之前的内核上，`conntrack`模块在插入条目时存在竞争条件，使用`NFQUEUE`机制导致`AAAA`请求包被丢弃，从而导致`DNS`请求出现`5`秒超时的现象。当时给的解决方案是可以在`/etc/resolv.conf`中添加`options single-request-reopen`来规避。

最近又遇到这个问题，但该规避方案并没有生效，因而做了进一步分析，发现`glibc`的`DNS`行为在不同响应上有所不同。

在没有`NFQUEUE`机制也没有开启`single-request-reopen`的环境中，访问正常解析的域名, 如:

```plain
[root@dev07 ~]# time curl -s www.baidu.com -o /dev/null

real	0m1.009s
user	0m0.003s
sys	0m0.003s
```

抓包结果为:
```plain
16:29:40.344792 IP 10.10.0.7.34972 > 10.10.0.2.53: 20510+ A? www.baidu.com. (31)
16:29:40.344832 IP 10.10.0.7.34972 > 10.10.0.2.53: 42033+ AAAA? www.baidu.com. (31)
16:29:40.345097 IP 10.10.0.2.53 > 10.10.0.7.34972: 20510 3/0/0 CNAME www.a.shifen.com., A 110.242.70.57, A 110.242.69.21 (90)
16:29:40.345308 IP 10.10.0.2.53 > 10.10.0.7.34972: 42033 3/0/0 CNAME www.a.shifen.com., AAAA 2408:871a:2100:1b23:0:ff:b07a:7ebc, AAAA 2408:871a:2100:186c:0:ff:b07e:3fbc (114)
```

可以看到`resolver`同时发出`A`和`AAAA`两个请求。

<!--more-->

开启`NFQUEUE`机制后，再次尝试，`DNS`延时`5`秒:
```plain
[root@dev07 ~]# time curl -s www.baidu.com -o /dev/null

real	0m5.556s
user	0m0.003s
sys	0m0.004s
```

抓包结果为:
```plain
16:31:05.061444 IP 10.10.0.7.41882 > 10.10.0.2.53: 35871+ A? www.baidu.com. (31)
16:31:05.061710 IP 10.10.0.2.53 > 10.10.0.7.41882: 35871 3/0/0 CNAME www.a.shifen.com., A 110.242.70.57, A 110.242.69.21 (90)
16:31:10.065884 IP 10.10.0.7.41882 > 10.10.0.2.53: 35871+ A? www.baidu.com. (31)
16:31:10.066183 IP 10.10.0.2.53 > 10.10.0.7.41882: 35871 3/0/0 CNAME www.a.shifen.com., A 110.242.70.57, A 110.242.69.21 (90)
16:31:10.066253 IP 10.10.0.7.41882 > 10.10.0.2.53: 61492+ AAAA? www.baidu.com. (31)
16:31:10.066398 IP 10.10.0.2.53 > 10.10.0.7.41882: 61492 3/0/0 CNAME www.a.shifen.com., AAAA 2408:871a:2100:1b23:0:ff:b07a:7ebc, AAAA 2408:871a:2100:186c:0:ff:b07e:3fbc (114)
```

抓包中未见`AAAA`请求，它是由于`conntrack`条目的冲突被内核丢弃，因而只收到了`A`请求的响应包，在`5`秒超时后，`resolver`开启单独发送`A`请求，获得响应包后，再次发送`AAAA`请求。

这种场景下，在`/etc/resolv.conf`中添加上`options single-request-reopen`，再次尝试, 问题得到解决:
```plain
[root@dev07 ~]# time curl -s www.baidu.com -o /dev/null

real	0m1.061s
user	0m0.001s
sys	0m0.005s
```

从抓包结果看，`resolver`不再同时发送`A`和`AAAA`请求而是单独发送，不会再触发`conntrack`条目冲突的BUG，因而可以正常解析。
```plain
16:36:41.257280 IP 10.10.0.7.34654 > 10.10.0.2.53: 38521+ A? www.baidu.com. (31)
16:36:41.257561 IP 10.10.0.2.53 > 10.10.0.7.34654: 38521 3/0/0 CNAME www.a.shifen.com., A 110.242.69.21, A 110.242.70.57 (90)
16:36:41.257694 IP 10.10.0.7.35329 > 10.10.0.2.53: 54925+ AAAA? www.baidu.com. (31)
16:36:41.257870 IP 10.10.0.2.53 > 10.10.0.7.35329: 54925 3/0/0 CNAME www.a.shifen.com., AAAA 2408:871a:2100:1b23:0:ff:b07a:7ebc, AAAA 2408:871a:2100:186c:0:ff:b07e:3fbc (114)
```

但是当`DNS`服务器响应`ServFail`时，`resolver`的行为和上述并不一致。我们使用`unbound`构造一个响应`ServFail`的环境。

在没有`NFQUEUE`机制也没有开启`single-request-reopen`的环境中，访问返回`ServFail`的域名, 很快就返回失败:
```plain
[root@dev07 ~]# time curl -s xxx.com -o /dev/null

real	0m0.009s
user	0m0.004s
sys	0m0.003s
```

抓包结果为:
```plain
16:47:26.232456 IP 10.10.0.7.39800 > 10.10.0.2.53: 50390+ A? xxx.com. (25)
16:47:26.232473 IP 10.10.0.7.39800 > 10.10.0.2.53: 18664+ AAAA? xxx.com. (25)
16:47:26.232719 IP 10.10.0.2.53 > 10.10.0.7.39800: 50390 ServFail 0/0/0 (25)
16:47:26.233144 IP 10.10.0.2.53 > 10.10.0.7.39800: 18664 ServFail 0/0/0 (25)
16:47:26.233193 IP 10.10.0.7.39800 > 10.10.0.2.53: 50390+ A? xxx.com. (25)
16:47:26.233202 IP 10.10.0.7.39800 > 10.10.0.2.53: 18664+ AAAA? xxx.com. (25)
16:47:26.233984 IP 10.10.0.2.53 > 10.10.0.7.39800: 50390 ServFail 0/0/0 (25)
16:47:26.234001 IP 10.10.0.2.53 > 10.10.0.7.39800: 18664 ServFail 0/0/0 (25)
```

`resolver`同时发送`A`和`AAAA`请求，然后收到两个`ServFail`的响应，然后立即重试，再次同时发送`A`和`AAAA`请求，收到两个`ServFail`响应后结束。

此时，开启`NFQUEUE`机制，再次尝试，出现`5`秒超时的现象:
```plain
[root@dev07 ~]# time curl -s xxx.com -o /dev/null

real	0m6.520s
user	0m0.002s
sys	0m0.004s
```

抓包结果为:
```plain
16:50:31.065725 IP 10.10.0.7.43428 > 10.10.0.2.53: 7203+ A? xxx.com. (25)
16:50:31.066012 IP 10.10.0.2.53 > 10.10.0.7.43428: 7203 ServFail 0/0/0 (25)
16:50:36.070160 IP 10.10.0.7.43428 > 10.10.0.2.53: 7203+ A? xxx.com. (25)
16:50:36.070186 IP 10.10.0.7.43428 > 10.10.0.2.53: 4659+ AAAA? xxx.com. (25)
16:50:36.073295 IP 10.10.0.2.53 > 10.10.0.7.43428: 4659 ServFail 0/0/0 (25)
16:50:37.165437 IP 10.10.0.2.53 > 10.10.0.7.43428: 7203 ServFail 0/0/0 (25)
```

同样，抓包未见`AAAA`请求，因为同时发出的`AAAA`请求由于`conntrack`条目冲突被内核丢弃。`resolver`只收到`A`请求的`ServFail`响应，没有收到`AAAA`请求的响应。在等待`5`秒后，再次同时发送`A`和`AAAA`请求，此时因为`conntrack`条目已经`confirmed`，因而不会再出现插入冲突，因而都能发送出去，收到两个`ServFail`后`DNS`请求结束。

此时，在`/etc/resolv.conf`中添加上`options single-request-reopen`，再次尝试, 问题并没有得到解决:
```plain
[root@dev07 ~]# time curl -s xxx.com -o /dev/null

real	0m10.524s
user	0m0.002s
sys	0m0.004s
```

抓包结果为:
```plain
16:56:44.096299 IP 10.10.0.7.35608 > 10.10.0.2.53: 7186+ A? xxx.com. (25)
16:56:44.096581 IP 10.10.0.2.53 > 10.10.0.7.35608: 7186 ServFail 0/0/0 (25)
16:56:49.100731 IP 10.10.0.7.35608 > 10.10.0.2.53: 7186+ A? xxx.com. (25)
16:56:49.100951 IP 10.10.0.2.53 > 10.10.0.7.35608: 7186 ServFail 0/0/0 (25)
```

`resolver`单独发送`A`请求，得到`ServFail`请求，这种情况下，并没有像同时发送`A`和`AAAA`得到两个`ServFail`响应的场景下立即重试，而是等待`5`秒后，再次重新发送`A`请求。

因而这种场景下，添加`single-request-reopen`并没能规避`NFQUEUE`机制引发的`DNS`超时现象。

在这种场景下的，简单的规避方案，是可以在`/etc/hosts`中添加上对应的域名信息，避免发送DNS请求。

如果要在不升级内核的场景下解决，可以尝试在通过`NFQUEUE`上送到用户态前，提前`confirm`该`conntrack`条目，这样可以规避掉`NFQUEUE`引发的`conntrack`冲突。
