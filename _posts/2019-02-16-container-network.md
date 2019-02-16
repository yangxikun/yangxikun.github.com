---
layout: post
title: "容器网络"
description: "容器网络"
category: Docker
tags: []
---
{% include JB/setup %}

本文学习总结自**极客时间 深入剖析Kubernetes**。

#### 单宿主机容器网络互通
- - -

容器通过Veth Pair连接到docker0网桥上，实现了容器与容器之间，容器与宿主机之间的网络互通。

* Veth: virtual Ethernet devices
* Veth Pair: 一对虚拟以太网设备，可以看作现实生活中的网线，能在不同namespace下传输流量

YouTube视频：[Linux VETH Pair - Virtual Ethernet Pair](https://www.youtube.com/watch?v=FUHyWfRNhTk)

2个容器之间直接通过Veth Pair可以实现网络互通，但如果有n个容器之间需要互通的话，就可能需要创建n(n-1)/2个Veth Pair，就跟现实生活中需要把很多台计算机实现网络互通一样。

在现实生活中，通常是采用网桥，只需要将每台计算机通过一根网线连接在网桥上，那么计算机之间就可以实现网络互通了。

所以可以通过在操作系统上虚拟出一个网桥设备，将容器通过Veth Pair连接到网桥上，实现容器之间网络互通。

YouTube视频：[Docker Advanced Networking](https://www.youtube.com/watch?v=Xxhhdo2e-DA)

![](/assets/img/201902160101.png)

使用bridge网络的容器，路由表类似这样子：

```plaintext
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```

对于172.17.0.0/16网段的路由是一条直连规则，凡是匹配到这条规则的IP包，应该经过eth0网卡，通过二层网络直接发往目的主机，因为当前主机和目的主机连接在同一个网桥上。而对于其他IP包，则需要发给网关，网关会根据自己的路由信息对数据包进行转发。

<!--more-->

#### 跨宿主机容器网络互通
- - -

Overlay Network（覆盖网络）：通过软件构建一个覆盖宿主机网络之上的、可以把所有容器连通在一起的虚拟网络。

![](/assets/img/201902160102.png)

##### Flannel UDP

![](/assets/img/201902160103.png)

flannel0：Tunnel设备，一种工作在三层的虚拟网络设备，在操作系统内核和用户应用程序之间传递IP包。

每台宿主机上的所有容器会被分配到一个子网中。

flanneld进程在处理fannel0传入的IP包时，就可以根据目的IP地址，匹配到对应的子网，找到这个子网对应的宿主机的IP地址，将IP包直接封装在UDP包里，发给对应的宿主机。

该模式的性能问题在于多了4次用户态和内核态之间的数据拷贝：

![](/assets/img/201902160104.png)

##### Flannel VXLAN

VXLAN：Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。VXLAN 可以完全在内核态实现UDP模式的封装和解封装的工作。

VTEP: VXLAN Tunnel End Point（虚拟隧道端点），在内核态进行封装和解封装，对象是二层数据帧。

![](/assets/img/201902160105.png)

VXLAN 模式组建的覆盖网络，其实就是一个由不同宿主机上的 VTEP 设备，也就是 flannel.1 设备组成的虚拟二层网络。对于 VTEP 设备来说，它发出的“内部数据帧”就仿佛是一直在这个虚拟的二层网络上流动。这，也正是覆盖网络的含义。

##### Flannel host-gw

在宿主机路由表里，将每个 Flannel 子网的“下一跳”，设置成了该子网对应的宿主机的 IP 地址。该模式要求集群宿主机之间是二层连通的。

![](/assets/img/201902160106.png)

##### Calico

不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico 项目使用了BGP来自动地在整个集群中分发路由信息。

BGP 的全称是 Border Gateway Protocol，即：边界网关协议。它是一个 Linux 内核原生就支持的、专门用在大规模数据中心里维护不同的“自治系统”之间路由信息的、无中心的路由协议。

自治系统，指的是一个组织管辖下的所有 IP 网络和路由器的全体。两个自治系统里的主机，要通过 IP 地址直接进行通信，我们就必须使用路由器把这两个自治系统连接起来。

![](/assets/img/201902160107.png)

边界网关：把自治系统连接在一起的路由器。它跟普通路由器的不同之处在于，它的路由表里拥有其他自治系统里的主机路由信息。

在使用了 BGP 之后，你可以认为，在每个边界网关上都会运行着一个小程序，它们会将各自的路由表信息，通过 TCP 传输给其他的边界网关。而其他边界网关上的这个小程序，则会对收到的这些数据进行分析，然后将需要的信息添加到自己的路由表里。

除了对路由信息的维护方式之外，Calico 项目与 Flannel 的 host-gw 模式的另一个不同之处，就是它不会在宿主机上创建任何网桥设备。

![](/assets/img/201902160108.png)

Calico 维护的网络在默认配置下，是一个被称为“Node-to-Node Mesh”的模式。这时候，每台宿主机上的 BGP Client 都需要跟其他所有节点的 BGP Client 进行通信以便交换路由信息。但是，随着节点数量 N 的增加，这些连接的数量就会以 N²的规模快速增长，从而给集群本身的网络带来巨大的压力。

所以，Node-to-Node Mesh 模式一般推荐用在少于 100 个节点的集群里。而在更大规模的集群中，你需要用到的是一个叫作 Route Reflector 的模式。

在这种模式下，Calico 会指定一个或者几个专门的节点，来负责跟所有节点建立 BGP 连接从而学习到全局的路由规则。而其他节点，只需要跟这几个专门的节点交换路由信息，就可以获得整个集群的路由规则信息了。

这些专门的节点，就是所谓的 Route Reflector 节点，它们实际上扮演了“中间代理”的角色，从而把 BGP 连接的规模控制在 N 的数量级上。

Calico 要求集群宿主机之间是二层连通的，如果要实现三层连通的宿主机之间的容器网络互联，就需要打开Calico IPIP模式。

![](/assets/img/201902160109.png)

Calico 使用的这个 tunl0 设备，是一个 IP 隧道（IP tunnel）设备。

在上面的例子中，IP 包进入 IP 隧道设备之后，就会被 Linux 内核的 IPIP 驱动接管。IPIP 驱动会将这个 IP 包直接封装在一个宿主机网络的 IP 包中，如下所示：

![](/assets/img/201902160110.png)

当 Calico 使用 IPIP 模式的时候，集群的网络性能会因为额外的封包和解包工作而下降。在实际测试中，Calico IPIP 模式与 Flannel VXLAN 模式的性能大致相当。所以，在实际使用时，如非硬性需求，建议将所有宿主机节点放在一个子网里，避免使用 IPIP。

避免使用 Calico IPIP 模式的方案：

1. 所有宿主机都跟宿主机网关建立 BGP Peer 关系，要求宿主机网关必须支持一种叫作 Dynamic Neighbors 的 BGP 配置方式。这是因为，在常规的路由器 BGP 配置里，运维人员必须明确给出所有 BGP Peer 的 IP 地址。考虑到 Kubernetes 集群可能会有成百上千个宿主机，而且还会动态地添加和删除节点，这时候再手动管理路由器的 BGP 配置就非常麻烦了。而 Dynamic Neighbors 则允许你给路由器配置一个网段，然后路由器就会自动跟该网段里的主机建立起 BGP Peer 关系。
1. 使用一个或多个独立组件负责搜集整个集群里的所有路由信息，然后通过 BGP 协议同步给网关。在大规模集群中，Calico 本身就推荐使用 Route Reflector 节点的方式进行组网。所以，这里负责跟宿主机网关进行沟通的独立组件，直接由 Route Reflector 兼任即可。这种情况下网关的 BGP Peer 个数是有限并且固定的。所以我们就可以直接把这些独立组件配置成路由器的 BGP Peer，而无需 Dynamic Neighbors 的支持。这些独立组件的工作原理也很简单：它们只需要 WATCH Etcd 里的宿主机和对应网段的变化信息，然后把这些信息通过 BGP 协议分发给网关即可。

#### 三层网络方案和"隧道模式"方案的异同
- - -

* 相同：都实现了跨宿主机容器的三层连通。
* 不同：三层网络方案通过配置下一跳主机的路由规则来实现连通，"隧道模式"方案通过在IP包外再封装一层UDP包来实现连通。
* 三层网络方案：Flannel host-gw、Calico
    * 优点：少了封包和解包的过程，性能高
    * 缺点：需要想办法维护路由规则
* "隧道模式"方案：Flannel UDP、Flannel VXLAN、Calico IPIP
    * 优点：简单，大部分工作可以由Linux 内核的模块实现
    * 缺点：性能低