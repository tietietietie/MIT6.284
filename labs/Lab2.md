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
* Term: 标志着“任期”，是随选举次数不断增长的整数，如果follower发现其term过小，则增大term，如果candidate或者leader发现其term过小（失效），则立刻变为follower

接下来分“选举”，“log复制"，"安全性”三部分讨论实现细节。

### Leader Selection

Follower：如果在一段时间（election timeout）没有收到RPC请求，则开始一段新的选举，并把自己当作候选人。

* 注意每个follower的timeout时间不一样（随机化），以保证不会有很多follower同时变为candidate，同时也保证新的leader能及时发送heartbeat

Candidate：follower增加term，进入新的选举周期，并把自己的状态改为candidate，并向其他服务器发送RequestVote请求。以下三种情况，candidate会发生状态转变

* canditate获得大部分选票（每个server按照先来后到的顺序，在一个选期（term）内进行唯一的一次投票），一旦成为leader，立刻向其余server发送心跳
* candidate收到了来自其余server的心跳，并在同一term，则自动变为follower。如果是过期的term，则忽略。
* 如果没有leader选出来，则自动开始下一轮拉票。（所有canditate的失效时间不同，大多数情况下，只有一个candidate先发生timeout，开始下一轮竞选，此时他会胜出并）



## Lab2A

目标：完成src/raft.go代码，实现选举single leader的功能。

待实现代码：

* RequestVoteArgs和RequestVoteReply的结构体
* 修改make()，能够阶段性的发送RequestVote RPC ？？
* 设置election timeout保证能在5秒内进行多次election （但是要大于300ms）
* 实现GetState()
* 实现DPrintf，进行debug

### 参考建议

#### Locking advice

* 变量被多个goroutine使用，必须加锁。
* 对一系列修改变量的操作进行统一，防止歧义。（如下，如果分开加锁或者对state不加锁，可能会造成curTerm和state不统一）

```go
  rf.mu.Lock()
  rf.currentTerm += 1
  rf.state = Candidate
  rf.mu.Unlock()
```

* 读取共享变量也有可能要加锁（如下，如果不对if语句加锁，在if执行完后，另一routinue操作curTerm使他变大,最终可能会造成curTerm减小）

```go
  rf.mu.Lock()
  if args.Term > rf.currentTerm {
   rf.currentTerm = args.Term
  }
  rf.mu.Unlock()
```

* 对需要等待的代码**谨慎加锁**，防止死锁。比如不必对RPC调用加锁。
* 总结：对于没有并发同步的变量不必加锁，对于有同步的变量，在go routine开始时加锁，结束时释放锁（暴力直接的方法）。对于需要等待的代码位置，谨慎加锁。

#### Raft Structure Advice

* 使用共享变量和锁来更新raft instance的状态（log, current index, &c)
* 将发送心跳（leader）以及进行选举（waited followers)的go routine分开写
* 把最近一次收到log的时间保存在state中，使用time.sleep周期性的检查时间间隔，以决定是否发生新一轮选举
* 使用Go的sync.Cond来出发一个单一的长期运行的goroutine
* 每一个RPC请求最好在各自的goroutine中发送，为了保证无法到达的raft设备不会延误大多数设备的请求收集工作，也为了保证发送的timer能够不断的运行
* 注意由于网络原因，你发送的RPC顺序不会有啥参考价值，leader必须检查reply中的term有没有改变，并且保证nextIndex相同（因为可能发送多条重复RPC？漏包？）

### Code

先整体看下大致结构，其中lab 2A需要完成以下工作：

* 定义部分的raft state，包括currentTerm, voteFor, leader.
* 实现GetState()（获得当前raft的状态，包括curTerm和isLeader）
* 实现RequestVoteArgs和RequestVoteReply结构体（请求投票RPC的参数及返回体）
  * Args
    * term：候选人当前的选举周期
    * 