---
title: Paxos 共识算法
date: 2020/8/13 13:59:0
tags: [分布式]
categories: [分布式]
---

共识算法的主要目的是屏蔽掉故障节点产生的噪音，从而让系统能够正常运行下去，常用于选举和状态机复制（state machine replication）。在保证 liveness 的同时，对 agreement 条件放宽了要求，它接受不一致是常态的事实，既然无法知道某些节点是 crash 还是消息延迟，那么只关心能够正确响应的节点，只要它们能够表决过半即可。过半表决意味着虽然没有达成完全一致，但是投票结果已经被过半的节点所继承，这样任何两个 quorum 一定会存在交集（比如 A、B、C 三个节点，两个 quorum 比如 AB 和 BC 一定会有交集 B），所以它们最终一定能通过消息交互而达成一致。

<!--more-->

Paxos 算法由 Lamport 提出，他为了描述该算法，虚拟出了一个叫做 Paxos 的希腊城邦，这个城邦按照民主投票的方式制定法令，但是没有人愿意将自己的全部时间和精力放在这上面，无论是提议者还是接收者亦或是信使，他们只会不定时地来参加提议。Paxos 算法的目的就是让他们按照少数服从多数的方式达成一致的意见。在他的论文中，Paxos 共有三种不同的变种：Basic Paxos、Multi Paxos 和 Fast Paxos。

# 角色
- Client
客户端，可以理解为普通民众，他们不直接参与提议，但是他们会希望通过一个对自己有利的法令。

- Proposer
提议者，可以理解为议员，他们接收 Client 的请求，代表民众向议会中的其他议员提出提议（Proposal），并在发生冲突时起到调节的作用。

- Acceptor
提议接收者，可以理解为议会中的其他议员，他们接收提议者提出的提议，只有接收到提议的成员达成法定人数（quorum，一般为 majority 多数派）时，提议才会形成法令。

- Learner
同样也是提议接收者，只不过他们并不参与提议过程，有点像议会记录人员，只负责记录已经通过的法令。

# Basic Paxos
Basic Paxos 分为两个大的阶段，同时每个大的阶段又分为两个小的阶段。

1. Phase 1a：Prepare
```
proposer             acceptor
   |                    |
   |     prepare(n)     |
   |------------------->|
```
proposer 向任意一个多数派 acceptor 集合发起一个编号 number 为 n 的预请求（prepare request）。

2. Phase 1b: Promise
```
proposer             acceptor
   |                    |
   |     prepare(n)     |
   |------------------->|
   |                    |
   |  <ok, null, null>  |
   |<-------------------|
```
如果一个 acceptor 接收到了预请求，并且预请求的编号 n 大于任何它之前已经回复过的请求（包含 prepare request 和 accept request），那么它将承诺不再接受任何编号小于 n 的提议（proposal），并回复已经接受的编号最大的提议（如果有的话）。

3. Phase 2a: Accept
```
proposer             acceptor
   |                    |
   |     prepare(n)     |
   |------------------->|
   |                    |
   |  <ok, null, null>  |
   |<-------------------|
   |                    |
   |    accept(n, v)    |
   |------------------->|
```
如果 proposer 接收到了来自 acceptor 对它预请求的回复，那么接下来它就可以向多数派 acceptor 集合（可以不是回复预请求的那个集合）发送一个 number 为 n，value 为 v 的提议（proposal）。其中的 v 要么是 proposer 自定义的值，要么就是预请求响应中的 value 值。我们将这样的一个请求称为 accept request。

4. Phase 2b: Accepted
```
proposer             acceptor
   |                    |
   |     prepare(n)     |
   |------------------->|
   |                    |
   |  <ok, null, null>  |
   |<-------------------|
   |                    |
   |    accept(n, v)    |
   |------------------->|
   |                    |
   |     <ok, n, v>     |
   |<-------------------|
```
如果 acceptor 接收到了 number 为 n 的 accept request，并且它没有对 number 大于 n 的 prepare request 进行过回复，那么就接受这个 accept request。

Basic Paxos 算法中还有很多细节，可能通过上述步骤并没有很好地展现出来，这里简单列举一下：

- acceptor 必须接受它收到的第一个 proposal。并且通过上述步骤可以发现，如果一个 value 为 v 的 proposal 被选中，那么所有被选中的拥有更大编号的 proposal 的 value 值也都是 v。

- acceptor 本地主要存储两个值，一个是已经接受的 proposal，另一个就是回复过的 prepare request 中最大的编号。

- acceptor 总是能够对 prepare request 进行回复，除非 acceptor 还没有接受一个 proposal，同时这个 prepare request 的编号小于 acceptor 已经回复过的 prepare request 中最大的编号。

- 与我们的常识不同，每个 proposer 并不会执着于自己的提议通过，而是会努力让一个提议尽快达成一致。

- 上面没有画出 learner，但是在 acceptor 接受了一个 proposal 之后，需要 acceptor 将这个 proposal 发送给所有的 learner，由 learner 来确定是否有一个 proposal 已经被多数派接受了，之后 learner 会将最终结果发送给 Client。

## Basic Paxos 的潜在问题
在 prepare request 阶段可能会出现活锁，即多个 proposer 不断拿着比其他人更大的 number 来发起 prepare request，导致算法一直停留在第一阶段，可以看出，这是一个关于 liveness 的问题。目前工业界的解决方案很简单，就是在发生冲突时，加入随机等待时间，在 timeout 之后再发起 prepare request。

Basic Paxos 还有一些其他问题，比如可能比较难以实现，同时由于需要两轮 RPC，导致效率较低。

# Multi Paxos
Basic Paxos 通过多轮的 prepare/accept 过程来确定一个值，我们可以称这整个过程为一个 instance。而如果需要确定很多个值，那么每个值的确定都需要经过一个 instance，显然效率不是很高。而 Multi Paxos 就是通过改变 promise 的生效范围至全局的 instance，使得我们之后可以省略第一阶段的工作（即使不 prepare 就直接 accept 也是安全的，因为 accept 已经被其中某个节点 promise 过了）。

上面的说法有些抽象，具体来说就是：首先也是有一些 proposer，但是依靠某种机制（比如 Raft 中的随机超时时间），proposer 集合中的某一个 proposer 自发地想成为 leader，此时它需要向 acceptor 集合中发起与 Basic Paxos 类似的两轮 RPC，在形成了多数派之后，此 proposer 才会正式成为 leader。接下来就通过 leader 来发起提议，并且不需要向 acceptor 集合发起 prepare request，直接发起 accept request 即可。

需要注意的是，Multi Paxos 是允许并行提交的，这就意味着可以不通过 leader 发起提议，但是这种情况下效率就会降低。比如在有一个 leader 的情况下，另一个 proposer 发起提议，就会导致 leader 的 accept 不被接受，此时就退化成了 Basic Paxos。因此我们希望大部分时间只有一个节点在提交，这样才能发挥 Multi Paxos 的优化效果。那么为了避免或者说是降低这种情况的发生，就需要一些控制：如果一个节点向其他节点发送 accept request，并且最后收到了来自其他节点的 accepted 回复，那么这个节点会认为自己是 leader，它会在一段时间内，拒绝其他节点的 prepare 请求。如果一个节点收到了来自其他节点的 accept request，那么它会认为请求方是 leader，并承诺在一段时间内不会发起 prepare request。正是这种“默契”才间接地说明了 leader 的存在，所以说在 Multi Paxos 中，leader 不是靠选举产生的。

# 参考
> [Paxos 理论介绍(2): Multi-Paxos 与 Leader](https://zhuanlan.zhihu.com/p/21466932)

> 《Paxos Made Simple》