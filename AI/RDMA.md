# 基于 RDMA 构建高速网络

无论是利用 overlay 实现的隧道网络，还是 underlay 方式的主机网络直通，其目的都是解决容器网络“通与不通”的问题。

大规模 AI 集群中，百亿、千亿级别参数量的大模型通常使用分布式训练，节点间需要交换海量的参数梯度等信息，若使用仅仅解决“通与不通”的入门级手段，对 AI 训练这种百亿级参数传递而言实在太慢了。

此时，就需要使用 RDMA 网络来传递。

## RDMA

RDMA（Remote Direct Memeory Access，程直接内存访问）是一种超高速的网络内存访问技术。

速度快的原因如下图所示，一次跨主机的网络请求，RDMA 可以绕过 TCP/IP 协议栈，并且不需要 CPU 干预，直接从网卡硬件上开始网络数据传递。RDMA 直接越过这些操作系统内核的开销，带来极为震撼的传输速率，大大加快训练参数的交换。

:::center
  ![](../assets/RDMA.png)<br/>
  图  RDMA 
:::

RDMA 的硬件主要由三种实现：Infiniband、RoCE 和 iWARP。其中，Infiniband、RoCE 是应用比较广泛的技术。

### Infiniband

从名字就能看出，Infiniband（无限带宽）性能最好。但价格也最贵，搭建基于 IB 技术的 RDMA 网络需要全套的网络设备支持（专用的网卡、交换机、线缆）。

### RoCE

为了实现近乎 IB 网络的高性能并降低成本，IBTA 定义了 RoCE（RDMA over Converged Ethernet），将 Infiniband 的四层传输协议 RDMA 移植到以太网上来。

RoCE 的发展其实涉及了 2 个版本。早期的 RoCEv1 仅支持在二层的以太网实现 RDMA 传输，由于场景和网
络规模受限（不能跨子网），并没有得到广泛的推广。直到 RoCEv2 出现，传输层的 RDMA 被封装在 UDP/IPv4 或
UDP/IPv6 的协议之上，实现了可以基于 3 层以太网路由的 RoCE 协议，解除了 RoCEv1 版本的应用限制。


:::tip ECN

ECN 是一种网络拥塞通知和管理的机制，它在监测到网络即将发生拥塞的时候，不会将报文丢弃，而是添加拥塞标记，让发送方动态调整拥塞控制窗口（CWND），从而控制拥塞。
:::

由于 RoCE 可以使用普通以太网交换设备，所以它在企业中应用比较广泛，

:::tip PFC

PFC（Priority-based Flow Control，基于优先级的流量控制）是目前应用最广泛的能够有效避免丢包的流量控制技术，是智能无损网络的基础。使能了 PFC 功能的队列，我们称之为无损队列。当下游设备的无损队列发生拥塞时，下游设备会通知上游设备会停止发送该队列的流量，从而实现零丢包传输。

:::

这个就需要同时在交换机和服务器RoCE网卡上，两侧同时配置PFC策略进行流控，以实现无损网络。
可见，「参数面」网络的管理，会比普通主机网络，多一份PFC调优的复杂度。