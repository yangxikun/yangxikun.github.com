---
layout: post
title: "Kubernetes 网络中数据包的流转"
description: "Kubernetes 网络中数据包的流转"
category: Kubernetes
tags: []
---
{% include JB/setup %}

本文基于 kubernetes 1.17.4 + calico IPIP 模式 + kube-proxy IPtables 模式，观测 pod 间数据包的流动。

#### 同宿主上 Pod 之间的流量

在 busybox-7d4f45df67-w2mvj 中，访问 nginx-7944498f44-7cllz。

首先查看 Pod 的路由表：

```text
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```
busybox-7d4f45df67-w2mvj 发送给 nginx-7944498f44-7cllz 的数据包会通过 eth0 发送给网关 169.254.1.1。

但网关 169.254.1.1 并不是实际存在的，calico 通过在宿主机上的 Veth Pair 上设置 proxy_arp，代理了 169.254.1.1 的 arp 请求，这样容器发送出去的所有数据包都会送到宿主机上的 Veth Pair。

calico 对于 169.254.1.1 的相关解释：

* [Why does my container have a route to 169.254.1.1?](https://docs.projectcalico.org/reference/faq#why-does-my-container-have-a-route-to-16925411)
* [Why can’t I see the 169.254.1.1 address mentioned above on my host?](https://docs.projectcalico.org/reference/faq#why-cant-i-see-the-16925411-address-mentioned-above-on-my-host)

查看 busybox-7d4f45df67-w2mvj 在宿主机上的 Veth Pair：

```text
➜  ~ calicoctl get workloadendpoint  yangxikun-k8s-busybox--7d4f45df67--w2mvj-eth0 -o yaml
apiVersion: projectcalico.org/v3
kind: WorkloadEndpoint
metadata:
  creationTimestamp: 2020-06-21T09:05:55Z
  generateName: busybox-7d4f45df67-
  labels:
    app: busybox
    pod-template-hash: 7d4f45df67
    projectcalico.org/namespace: default
    projectcalico.org/orchestrator: k8s
    projectcalico.org/serviceaccount: default
  name: yangxikun-k8s-busybox--7d4f45df67--w2mvj-eth0
  namespace: default
  resourceVersion: "1966670"
  uid: e56e7d6f-cab2-4b2a-96a5-1c8f908e567b
spec:
  endpoint: eth0
  interfaceName: cali84b7d149337
  ipNetworks:
  - 10.200.77.220/32
  node: yangxikun
  orchestrator: k8s
  pod: busybox-7d4f45df67-w2mvj
  profiles:
  - kns.default
  - ksa.default.default
```

从以上信息中可以知道 busybox-7d4f45df67-w2mvj 在宿主机上的 Veth Pair 为 cali84b7d149337，查看其 proxy_arp 的设置： 

```text
➜  ~ cat /proc/sys/net/ipv4/conf/cali84b7d149337/proxy_arp
1
```

现在数据包出现在了宿主机上，在宿主机上观察请求数据包和响应数据包的流动。

<!--more-->

##### PREROUTING 链的处理

raw 表的处理：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241250] TRACE: raw:PREROUTING:rule:2 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241257] TRACE: raw:cali-PREROUTING:rule:1 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241263] TRACE: raw:cali-PREROUTING:rule:2 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241268] TRACE: raw:cali-PREROUTING:return:5 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241273] TRACE: raw:PREROUTING:policy:3 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
```

raw 表命中的规则（省略掉了没命中的规则）：

1. Chain PREROUTING: `cali-PREROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
    1. Chain cali-PREROUTING: `MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:XFX5xbM8B9qR10JG */ MARK and 0xfff0ffff`
    1. Chain cali-PREROUTING: `MARK       all  --  cali+  *       0.0.0.0/0            0.0.0.0/0            /* cali:EWMPb0zVROM-woQp */ MARK or 0x40000`
    1. Chain cali-PREROUTING: return 5
1. Chain PREROUTING: policy ACCEPT

> 从 cali+ 进来的数据包会打上标记 0x4000

mangle 表的处理：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241284] TRACE: mangle:PREROUTING:rule:1 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241290] TRACE: mangle:cali-PREROUTING:rule:3 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241296] TRACE: mangle:cali-from-host-endpoint:return:1 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241301] TRACE: mangle:cali-PREROUTING:return:5 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241305] TRACE: mangle:PREROUTING:policy:2 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
```

mangle 表命中的规则：

1. Chain PREROUTING: `cali-PREROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:6gwbT8clXdHdC1b1 */`
    1. Chain cali-PREROUTING: `cali-from-host-endpoint  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-from-host-endpoint: 空链，return 1
    1. Chain cali-PREROUTING: return 5
1. Chain PREROUTING: policy ACCEPT

nat 表的处理：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241311] TRACE: nat:PREROUTING:rule:1 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241322] TRACE: nat:cali-PREROUTING:rule:1 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241330] TRACE: nat:cali-fip-dnat:return:1 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241338] TRACE: nat:cali-PREROUTING:return:2 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241342] TRACE: nat:PREROUTING:rule:2 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241353] TRACE: nat:KUBE-SERVICES:return:12 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241357] TRACE: nat:PREROUTING:policy:3 IN=cali84b7d149337 OUT= MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
```

nat 表命中的规则：

1. Chain PREROUTING: `cali-PREROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
    1. Chain cali-PREROUTING: `cali-fip-dnat  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-fip-dnat: 空链，return 1
    1. Chain cali-PREROUTING: return 2
1. Chain PREROUTING: `KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
    1. Chain KUBE-SERVICES: 没有匹配到任何规则，return 12
1. Chain PREROUTING: policy ACCEPT

##### 路由决策，FORWARD 链的处理：

mangle 表：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241367] TRACE: mangle:FORWARD:policy:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
```

mangle 表命中的规则：

1. Chain FORWARD: policy ACCEPT

filter 表：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241372] TRACE: filter:FORWARD:rule:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241379] TRACE: filter:cali-FORWARD:rule:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x40000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241384] TRACE: filter:cali-FORWARD:rule:2 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241394] TRACE: filter:cali-from-hep-forward:return:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241399] TRACE: filter:cali-FORWARD:rule:3 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241406] TRACE: filter:cali-from-wl-dispatch:rule:3 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241419] TRACE: filter:cali-fw-cali84b7d149337:rule:3 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241428] TRACE: filter:cali-fw-cali84b7d149337:rule:6 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241442] TRACE: filter:cali-pro-kns.default:rule:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241452] TRACE: filter:cali-pro-kns.default:rule:2 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241461] TRACE: filter:cali-fw-cali84b7d149337:rule:7 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241467] TRACE: filter:cali-FORWARD:rule:4 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241479] TRACE: filter:cali-to-wl-dispatch:rule:2 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241492] TRACE: filter:cali-tw-cali5cb173fd42f:rule:3 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241580] TRACE: filter:cali-tw-cali5cb173fd42f:rule:4 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241597] TRACE: filter:cali-pri-kns.default:rule:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0
Jun 22 12:47:55 yangxikun kernel: [ 1713.241607] TRACE: filter:cali-pri-kns.default:rule:2 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241620] TRACE: filter:cali-tw-cali5cb173fd42f:rule:5 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241625] TRACE: filter:cali-FORWARD:rule:5 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241636] TRACE: filter:cali-to-hep-forward:return:1 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241642] TRACE: filter:cali-FORWARD:rule:6 IN=cali84b7d149337 OUT=cali5cb173fd42f MAC=ee:ee:ee:ee:ee:ee:32:bb:37:fe:19:df:08:00 SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
```

filter 表命中的规则：

1. Chain FORWARD: `cali-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
    1. Chain cali-FORWARD: 标记为 0x0 `MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:vjrMJCRpqwy5oRoX */ MARK and 0xfff1ffff`
    1. Chain cali-FORWARD: `cali-from-hep-forward  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:A_sPAO0mcxbT9mOV */ mark match 0x0/0x10000`
        1. Chain cali-from-hep-forward: 空链，return 1
    1. Chain cali-FORWARD: `cali-from-wl-dispatch  all  --  cali+  *       0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-from-wl-dispatch: `cali-fw-cali84b7d149337  all  --  cali84b7d149337 *       0.0.0.0/0            0.0.0.0/0           [goto]`
            1. Chain cali-fw-cali84b7d149337: `MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:vGWW-lGiGDypzTZK */ MARK and 0xfffeffff`
            1. Chain cali-fw-cali84b7d149337: `cali-pro-kns.default  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
                1. Chain cali-pro-kns.default: 标记为 0x10000 `MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:oLzzje5WExbgfib5 */ MARK or 0x10000`
                1. Chain cali-pro-kns.default: `RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:4goskqvxh5xcGw3s */ mark match 0x10000/0x10000`
            1. Chain cali-fw-cali84b7d149337: `RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:xz2hA-jMqP1hAU4H */ /* Return if profile accepted */ mark match 0x10000/0x10000`
    1. Chain cali-FORWARD: `cali-to-wl-dispatch  all  --  *      cali+   0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-to-wl-dispatch: `cali-tw-cali5cb173fd42f  all  --  *      cali5cb173fd42f  0.0.0.0/0            0.0.0.0/0           [goto]`
            1. Chain cali-tw-cali5cb173fd42f: `MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:Y_BDzGsQR3McUMEq */ MARK and 0xfffeffff`
            1. Chain cali-tw-cali5cb173fd42f: `cali-pri-kns.default  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
                1. Chain cali-pri-kns.default: 标记为 0x10000 `MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:7Fnh7Pv3_98FtLW7 */ MARK or 0x10000`
                1. Chain cali-pri-kns.default: `RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:ZbV6bJXWSRefjK0u */ mark match 0x10000/0x10000`
            1. Chain cali-tw-cali5cb173fd42f: `RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:-5aED4ednrmsLeOA */ /* Return if profile accepted */ mark match 0x10000/0x10000`
    1. Chain cali-FORWARD: `cali-to-hep-forward  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-to-hep-forward: 空链，return 1
    1. Chain cali-FORWARD: `ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:MH9kMp5aNICL-Olv */ /* Policy explicitly accepted packet. */ mark match 0x10000/0x10000`

##### POSTROUTING 链的处理

mangle 表：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241647] TRACE: mangle:POSTROUTING:policy:1 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
```

mangle 命中的规则：

1. Chain POSTROUTING: 空链，policy ACCEPT

nat 表：

```text
Jun 22 12:47:55 yangxikun kernel: [ 1713.241651] TRACE: nat:POSTROUTING:rule:1 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241658] TRACE: nat:cali-POSTROUTING:rule:1 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241664] TRACE: nat:cali-fip-snat:return:1 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241706] TRACE: nat:cali-POSTROUTING:rule:2 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241719] TRACE: nat:cali-nat-outgoing:return:2 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241725] TRACE: nat:cali-POSTROUTING:return:4 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241728] TRACE: nat:POSTROUTING:rule:2 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241731] TRACE: nat:KUBE-POSTROUTING:return:2 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
Jun 22 12:47:55 yangxikun kernel: [ 1713.241734] TRACE: nat:POSTROUTING:policy:3 IN= OUT=cali5cb173fd42f SRC=10.200.77.209 DST=10.200.77.216 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=23571 DF PROTO=ICMP TYPE=8 CODE=0 ID=6656 SEQ=0 MARK=0x10000
```

nat 表命中的规则：

1. Chain POSTROUTING: `cali-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
    1. Chain cali-POSTROUTING: `cali-fip-snat  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-fip-snat: 空链，return 1
    1. Chain cali-POSTROUTING: `cali-nat-outgoing  all  --  *      *       0.0.0.0/0            0.0.0.0/0`
        1. Chain cali-nat-outgoing: 没有匹配到任何规则，return 2
    1. Chain cali-POSTROUTING: return 4
1. Chain POSTROUTING: `KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0标记为 0x10000`
    1. Chain KUBE-POSTROUTING: 没有匹配到任何规则，return 2
1. Chain POSTROUTING: policy ACCEPT

响应包的流程与请求包的流程类似，除了不会经过 nat 表。

![](/assets/img/kubernetes-same-node-packet-flow.png)

#### 不同宿主上 Pod 之间的流量

![](/assets/img/kubernetes-nodes-packet-flow.png)

#### 通过 service 的流量

service 会有对应的 iptables 规则，在 nat 表 PREROUTING 子链上：

```text
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.200.0.0/16        10.32.0.139          /* default/nginx: cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-4N57TFCL4MD7ZTDA  tcp  --  *      *       0.0.0.0/0            10.32.0.139          /* default/nginx: cluster IP */ tcp dpt:80

Chain KUBE-MARK-MASQ (11 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-SVC-4N57TFCL4MD7ZTDA (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-3O5Y7HAHSBKY7LNO  all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-FQQ4PE7RRPPGGVIW  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain KUBE-SEP-3O5Y7HAHSBKY7LNO (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.200.77.229        0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.200.77.229:80

Chain KUBE-SEP-FQQ4PE7RRPPGGVIW (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.200.77.254        0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.200.77.254:80
```

每个 KUBE-SVC-* 对应一个 service，每个 KUBE-SEP-* 对应一个 Endpoint（Pod）。

![](/assets/img/kubernetes-nodes-svc-packet-flow.png)
