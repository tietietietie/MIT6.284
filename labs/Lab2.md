# Lab2 Raft

## 引言

Raft是复制状态机协议，通过将client请求处理位序列（log），并确保所有副机都能按相同顺序指定相同的log。从而确保副机和主机能提供相同的服务。Raft会处理服务器故障，确保恢复故障的服务器其log是同步的。Raft只有在多数服务器存活时才能使用？下面这一段不懂。

> Raft will continue to operate as long as at least a majority of the servers are alive and can talk to each other. If there is no such majority, Raft will make no progress, but will pick up where it left off as soon as a majority can communicate again.

在本次lab实验中，需要使用Go来实现Raft，raft被设计为一个对象，以及有配套的method。raft实例通过RPC进行通讯，并支持一个无限的log entries。

参考资料：

阅读[Paxos的大致原理](https://www.zhihu.com/question/19787937) ---> 阅读[Raft论文](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf) ---> 阅读[Bolosky论文](https://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)

### Paxos

选举过程，目的为尽可能达到一致，使意见通过

### Raft

比Paxos简单，更容易理解。体现在以下方面

* 将选举过程分解（一致性分解）：1）leader选出 2）log复制 3）安全性 4）成员变换
* 保证强一致性以减少需要同步的状态。

在外界看来，复制状态机是单一的一台高可靠服务器，在复制状态机内部，由**一致性模块**来保证服务器集群能够以相同的状态，按照相同的顺序执行相同的命令。

Raft便是保证log一致性的算法。

#### 术语解释

* Leader:处理所有来自client的请求，负责复制，发放Log，系统中只存在一个。
* Follower: passive，只会response来自Leader或者candidates的请求
* Candidate: 候选人，由follower转换而成，调用”收集选票"RPC
* Term: 标志着“任期”，是随时间连续增长的整数，如果follower发现其term过小，则增大term，如果candidate或者leader发现其term过小（失效），则立刻变为follower

接下来分“选举”，“log复制"，"安全性”三部分讨论实现细节。

## Lab2A

目标：完成src/raft.go代码，实现选举single leader的功能。

