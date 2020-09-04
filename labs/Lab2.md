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

* 定义部分的raft state，包括currentTerm, voteFor, leader， state（三种状态），log[]，
* 实现GetState()（获得当前raft的状态，包括curTerm和isLeader）
* 实现RequestVoteArgs和RequestVoteReply结构体（请求投票RPC的参数及返回体）
  * Args
    * term：候选人当前的选举周期
    * candidateId:候选人的ID
  * reply
    * term:候选人需要更新的term
    * voteGranted：true表示收到选票
* 实现RequestVote函数
  * labrpc包中的rpc.go中已经实现call函数，能够处理如server不可达，超时等情况（都会return false)，不需要自己在定义计时器。
  * 根据structure advice，为每一个request调用开一个goroutine
  * 向每一个server的port发送请求然后统计信息即可。
  * 接收者需要更新lastRequestTime
* 实现make函数，初始化raft
  * 初始化自己的state为follower
  * 初始化上次收到request的时间lastRequestTime为现在时间
  * 初始化自己的election timeout(random)
* 实现周期性检查(timeoutCheck（)）
  * 大概500ms一次？来检查rf的lastRequestTime
  * 如果超时，则把自己的状态改为candidate，并向所有的peer发送通知
* 定义logEntry结构体，发送空的就就可以。
* 定义AppendEntries的结构体
  * args
    * term
    * leaderId
    * entries[],对于heardbeat来说为空g'i't
  * reply
    * term,来自peer的term，需要更新
    * true，表示成功收到
* 定义AppendEntries handler，
  * 添加到log[]
  * 更新lastRequestTime

#### 实现RequestVote RPC

* 如果超过一定时间没有收到heartbeat，则term++,并转换为candidate
* candidtate把自己的ID以及term传入args，调用其余peers的RequestVote,获得结果。

#### 实现周期性检查

* 根据发送心跳频率，将server的term过期时间设置为500 - 800 ms
* raft初始化时自带一个rand数（500 - 800），并将lastRequestTime设置为now
* 周期性的检查自己的term是否有效，用一个长期运转的gorutine，每100ms获得一次锁
* 如果不是leader且过期，则把自己变为Candidate，并给自己投一票
* 如果收到大部分选票，则变为leader,并立刻发送心跳
* 如果没有收到大部分选票，则自动term增加

#### 实现发送心跳功能

* 周期性的发送(调用AppendEntries)，100ms一次
* 返回值为term和success，如果发现peer的term大于leader的tern，则把自己状态设置为follower并退出routine
* 收到Leader的心跳，如果term是合理的，则更新lastRequestTime和term，并返回true
* 如果收到了过期的心跳，则把自己的term发送过去，并返回false

#### 修BUG

* 通过运行 go test -race，发现rf.lastRequestTime存在竞争访问，原来忘记加defer了
* 通过查看DPrintf发现

![image-20200810193108615](D:\OneDrive\Study\CS\MIT6.824\labs\Lab2.assets\image-20200810193108615.png)

* 修bug难点，不停的添加DPrintf来看中间运行结果

1，统计选票不能在RegularCheck()函数中进行，RegularCheck只负责定期检查当前server是否最近收到了request，如果不是leader并且没有收到request，则进入选举

2，统计选票在sendRequestVote()中进行，如果本身状态已经是leader，则不统计选票，如果已经获得多数选票，则修改状态并立即发送心跳（另外开goroutine），如果term已经落后，变为follower，更新term和lastRequestTime。

3，旧的leader收到了新leader的心跳，应该把自己的状态设置为follower（也可以在发送心跳的时候设置）

最终通过实验

![image-20200811171632687](D:\OneDrive\Study\CS\MIT6.824\labs\Lab2.assets\image-20200811171632687.png)

#### 细节修改
* 每次过期后，需要重新选一个随机过期时间
* 只有在接收到AppendEntries或者**同意投票**后，才会重置astRequestTime

### 总结

多利用DPrintf，在疑似bug的地方把状态打印出来。

## Lab2B

### 设计目标
* 实现Start()：使用Raft()的service使用此函数传递命令给leader
* 实现AppendEntries()，发送和接收新的log
* 添加election限制条件

### 待完成代码
* 完成logEntry的定义：logEntry包括三个值：term, index, command
* 补充raft server的状态：commitIndex：已经提交的最高index， lastApplied：已经执行的最高index， nextIndex[]：表示每个follower当前发送的log index, matchIndex[]：表示已知的每个follower已经复制好的最高log index
* 添加election的限制条件：比较lastLogIndex和lastLogTerm，如果log不满足up-to-date,则不会投票给他
* 完成AppendEntries()的args和reply：
* 完成AppendEntries的handler代码：
leader:
收到client请求 ---> 添加到自己的log[] ---> 并发的向其余follower发送副本，直到大多数follower返回true ---> commitIndex + 1, lastApplied + 1
follower:
收到AppendEntries RPC ---> 比较term ---> 找到preLogIndex和preLogTerm的entry(否则返回false) ---> 存在冲突的entry，则删除此entry以及后面所有的entry ---> 更新commitIndex（根据leaderCommit和最新的entry index）
所有server:
每次commitIndex修改之后，需要比较commitIndex和lastApplied
commitIndex：
对于leader,只有收到了大部分follower的肯定，commitIndex才会加1，对于follower，他无法判断哪些log是commited，只有通过比较leaderCommit来更新
matchIndex:
表示这个follower从这个Index及之前，log都是相同的
nextIndex:
表示将要发送给这个follower的下一个index
会出现leader产生后，其commitIndex + 1也就是待提交的entry属于上一term，此时leader无法判断是否已经复制完成，但是raft不会去管这部分信息，而是先提交新term的appendEntris，来间接处理（便于立刻更新follower的lastTerm)

### 代码
* 定义logEntry结构体
* 补充raft状态
* 增加election限制条件: 修改RequestVote args和handler
* 完成AppendEntries()
  * 完成args和reply
  * handler：判断term ---> 判断是否为心跳 ---> 判断prev是否存在（不存在，false,nextIndex = lastLogIndex + 1) ---> 判断term是否相同，相同，则添加entries，true, nextLogIndex = lastLogIndex + 1 ---> 没有匹配上，false， nextIndex指向这个term的第一个index（或者committedIndex + 1）位置
* 修改heartBeat
* 处理command流程,即完成start函数：判断接收者是否为leader ---> 是leader，接受这个command，并产生entry，添加在leader log的后面
* 添加entries流程（选举为leader后就启动）（周期性检查添加,150ms） 判断nextIndex与lastLogIndex的关系，满足nextIndex <= lastLogIndex则添加 ---> 启动对应的sendAppendEntries,并启动计时器（100ms) ---> 构造args和reply ---> 如果超时，退出，如果收到reply，则判断 ---> true ? 更新nextIndex和matchedIndex : 更新nextIndex = reply.Index  ---> 150ms后，更新commitedIndex
* elction时，需要重置nextIndex和matchedIndex

### 修bug
* LogEntry里面的参数也要大写，因为要用于rpc传递
* 修改Dprintf，定位bug更方便
* applyCh：在commitIndex更新的地方使用（改完这个后终于通过了两个test）（打死不看注释系列）
* 修改HeartBeat机制，不能对HeartBeat特殊对待，也要对PrevLogTerm和PrevLogIndex进行检查。
* 修改for - sleep结构为timer结构
> Test (2B): no agreement if too many followers disconnect ...
panic: runtime error: index out of range [2] with length 2

goroutine 5004 [running]:
_/home/zhangtie/MIT6.824-Labs/src/raft.(*Raft).AppendEntries(0xc00026a700, 0xc0007f0e40, 0xc00042cc20)
        /home/zhangtie/MIT6.824-Labs/src/raft/raft.go:240 +0x554
越界错误：