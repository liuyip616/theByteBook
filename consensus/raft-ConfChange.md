# 6.4.3 成员变更

笔者前面讨论的部分假设的条件是 Raft 集群成员恒定不变，单实际上，我们会遇到很多需要改变数据副本数的情况，例如服务器故障需要移除副本，也就是说集群成员是需要动态变更的。

成员变更也是一个一致性问题，即所有节点对新成员达成一致，直接向 Leader 节点发送成员变更请求，Leader 同步成员变更日志，达成多数派之后提交，各节点提交成员变更日志后从旧成员配置（Cold）切换到新成员配置（Cnew）。但成员变更的一致性存在一个特殊性，

:::tip 问题背景
成员变更的主要问题在于在成员变更时，可能在某一时刻出现 Cold 和 Cnew 中同时存在两个不相交的多数派，进而可能选出两个 Leader，形成不同的决议，破坏安全性
:::


<div  align="center">
	<img src="../assets/raft-ConfChange.png" width = "350"  align=center />
	<p>成员变更的某一时刻 Cold 和 Cnew 中同时存在两个不相交的多数派</p>
</div>

如上图所示，3 个节点的集群扩展到 5 个节点，直接扩展可能会造成 Server1 和 Server2 构成老成员配置的多数派，Server3、Server4 和 Server5 构成新成员配置的多数派，两者不相交从而可能导致决议冲突。

由于成员变更的这一特殊性，一次成员变更不能当成一般的一致性问题去解决。为了解决这个问题，Raft 提出了两阶段的成员变更方法 Joint Consensus。

使用单节点变更方式很容易枚举出所有情况，如下图所示，穷举集群奇/偶数节点下添加和删除一个节点的情况，如果每次只增加和删除一个节点，那么 Cold 的 Majority 和 Cnew 的 Majority 之间一定存在交集，也就说是在同一个 Term 中，Cold 和 Cnew 中交集的那一个节点只会进行一次投票，要么投票给 Cold，要么投票给 Cnew，这样就避免了同一 Term 下出现两个 Leader。

<div  align="center">
	<img src="../assets/raft-single-server.png" width = "550"  align=center />
	<p>穷举集群添加节点的情况</p>
</div>

实际上除了 还有 Joint Consensus（联合共识），但这种方式实现起来很复杂，绝大多数的 Raft 算法采用的都是单节点变更方式，例如 Etcd、Hashicorp Raft 等。所以笔者在这里不再赘述，有兴趣的读者可以阅读 Diego Ongaro 的论文。