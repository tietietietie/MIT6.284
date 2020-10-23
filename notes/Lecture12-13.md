# Lecture12

## Paper: Atomicity

并发一致的要求一般有两种：顺序一致或原子性

顺序一致：可以通过显式的编程保证：比如“读取键盘的输入”一定发生在“键盘输入显示在屏幕”之前

### Before-or-After Atomicity

并发行为具有原子性是指：从其调用者的角度，该行为要么还没发生，要么已经发生，不存在中间状态。

优势：在不知道并发行为的发生顺序时使用。

没有确保原子性的典型错误例子：（注意写入操作通常有两部，读取数据 ---> 数据写入）

### 正确性和序列化

如何定义并发行为是正确的？等价于可序列化操作

>计算机系统对并发[事务](https://baike.baidu.com/item/事务/5945882)中并发操作的调度是随机的，而不同的调度可能会产生不同的结果。在计算机中，多个事务的[并发](https://baike.baidu.com/item/并发/11024806)执行是正确的，当且仅当其结果与按某一次序串行地执行它们时的结果相同，我们称这种[调度](https://baike.baidu.com/item/调度/206795)策略为可串行化（Serializable）调度。

什么是可序列化：一定顺序的并发事务发生后，产生了相同的结果，上图中的RWRW就是可以序列化并发事务。而RRWW就不是。

有时候需要比可序列化更强的正确性定义：必须必须保证事务T1发生在事务T2之前。

### 简单锁和两段锁

简单锁（一次性锁）是指transaction开始前，把**一次性把所有**可能要读写的数据都获得锁，再进行操作，commit或者roll back之后，释放锁。

两段锁把事务内部的锁分为两端，生长阶段（加锁阶段）和衰减阶段（解锁阶段），两阶段不能重合。一般加锁阶段在transaction开始执行时，解锁阶段在comit或者roll back之后。

分为S锁（共享锁）和X锁（排他锁），对只读数据加S锁，允许其他事务读，对要写数据加X锁，不允许其他事务读写。（生长阶段可以对锁升级）

### 分布式事务原子性： Two-Phase Commit（2PC）

背景：在分布式系统中，如果一项事务需要保证其原子性，如何实现？

2PC协议：保证分布式系统事务的原子性，分为事务协调者（TC）和事务管理器（TM，在单机上实现分布式事务的一部分）。分为两个阶段prepare和commit

Prepare阶段：TC写本地日志（WAL）并持久化，发送prepare给各个TM，TM写日志并持久化，决定提交发送Ready T，不准备提交发送abort T。

Commit阶段：TC发现所有TM都是ready状态，则写持久化日志，并发送commit命令，如果有一个TM时abort，则向所有TM发送abort命令。

几个要注意的点：

* TM（参与者）在发送Ready T之前，可以自由决定是否执行transaction，但是一旦发送，该transaction执行与否则完全交由TC决定。为了保证参与者一定能够执行这个承诺，需要WAL。
* 2PC一共需要发送3N次RPC请求
* 系统速度限制：disk write/ 多轮RPC， 所有通常运用在小型的局域系统，不适合银行，飞行航线等
* 优化思路：协调者不再持久化日志，而是让参与者持久化参与者名单及事务状态，从而降低持久化时间。另一个优化思路是，当所有TM都prepare后，TC直接返回成功，但可能增加读的等待时间（等待第二阶段的提交）
* 异常处理：
  * TM宕机，在发送ready前宕机，TC当作其abort，发送后宕机，忽略它是否宕机.
  * TM在事务执行时宕机,如果没有WAl,则放弃,如果有WAL,则需要向TC或者其他TM询问此次事务是否是commit,此时如果TC和其他TM无法联系,永久询问(不理想情况)
  * TC宕机,如果有一个TM中的该事务是已经commit,其他TM也会commit,有一个TW是abort,则其他事务也abort,如果全部TW的该事务都是ready,则只能等TC恢复(不理想情况,因为此时TW无法知道该事务的状态）
  * 换句话说，一但TM处于ready状态，但是一直没有收到TC的commit命令，则会一直等待，也会一直持有锁，整个系统变慢

Raft和2PC解决的问题不同：
Raft：解决avaibility，保证一个副本系统，大多数服务器都做相同的事，即便少量服务器宕机也能正常运行
2PC：解决atomicity,保证一项事务分割在不同的服务器执行时，也能有acid特性。

# Lecture13

## Paper: Spanner

### 一句话总结

Spanner是谷歌提出的**全球范围**的分布式数据库，支持**外部一致性**的分布式事务，通过独特的time API，能够支持lock-free的read-only事务，提高读取数据速度。

待看



## Video：Spanner

设计目的：实现**全球范围内**的分布式数据库（整个internet)，使得每一个人都能快速的访问到附近的副本，还能保证事务/容错。

基本思想：2PC分布式事务提交 + paxos副本容错 + 校准时间机制提速r/x xactions。

用户请求的强一致性：可序列化

分布式事务的外部一致性：t1 commit之后，t2开始时一定能看到t1造成的修改

服务架构：副本数据分布在全球各地的DataCenter中，同时一份data被切割成多份存储（存储在DC的不同服务器上）（便于同步）

一个Paxos集群管理的一份data切片，所以存在多个Paxos。由于存储在多个DC，用户可以从较近DC读取数据（提高速度）。

由此带来的两个问题：

1，Paxos会导致用户可能读到过期数据。（majority弊端）

2，由于数据分片，导致分布式事务。

### r/w xactions过程

Begin

​	x = x + 1

​	y = y - 1

End

x和y被切片，且都保存有三份副本在DC1, DC2, DC3。x和y都是有paxos leader管理的，有三个副本，但是我们可以看成一个整体。

1，client从两个paxos leader中获得x read-lock, y read-lock

2， 选择一个paxos group作为Transaction coordinater（TC），

3，client发送wx给paxos leader1，并且当paxos的将日志持久化后，回复yes给TC

4，同样将wy发送给paxos leader2, 也回复yes

5，如果都是yes，则TC commit此次事务，并把结果都通过Paxos用日志记录下来，然后将结果告诉给client和其他的paxos leader

6，执行write操作。

2pc Lock保证的可序列化，2pc保证了分布式事务的原子性。

spanner在这里解决了2pc的阻塞问题，因为TC本身是Paxos gruop，同时注意这里实现一次事务需要很多信息传输。





