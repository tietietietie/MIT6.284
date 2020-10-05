# Lab3

* 设计目标：实现可容错（复制）的K/V存储服务。
* API：Key/Value均为String
  * Put(key, value)：替换key对应的值为value
  * Append(key, arg)：key对应的value值后面添加arg，key不存在时，等同于Put
  * Get(key)：返回Key值对应的value，不存在时返回empty string
* 性能：提供强一致性

## Lab3A

* Client如何将操作发送给Server：通过一个Clerk代理，Clerk尝试与不同server进行RPC调用，直到找到leader.
* K/V servers之间是否需要通信？ 不需要，通过底层的raft实现一致性即可，server只需要执行applyCh

### 需要实现的代码

* Client端和Server端的RPC结构
* 定义Op结构体，来描述Put/Append/Get操作。（当Op被commit时，便可以返回RPC调用了）
* Start()：传入了一个Op(一个Log?)，开始达成一致性

### 难点

* 保证每一个Clerk op只被执行一次：给op分配一个unique ID
* Leader failure问题
  * old leader如何发现自己失去了leadership？自己的term已经改变？或者当前Index出出现了新的request
  * leader在提交了此次commit后failure了，如何保证clerk不会重复提交request？通过比较LastApplyedEntry的opID与此次的opID。

### 代码

**Clerk Struct**: 包括所有K/V server的地址，以及最新处理完成的opID，以及上次RPC时连接的leaderID，以及锁（保证opID的原子性更新），以及这个clerk所对用的client。

**GetArgs**:除了Key之外，还需要唯一的opID和clientID。

**Get**:第一次尝试连接的serverID初始化为上次RPC成功的leaderID，~~RPC的timeout时间设置为40ms~~，根据reply.Err判断是否操作成功。不需要判断RPC是否超时，因为RPC本身能够保证。（通过返回的bool ok）

**PutArgs**：与GetArgs类似。

**AppendPut RPC代码类似**

**KVServer Struct**:定义一个k/value表格（map)

### 修Bug

* 根据打印的日志发现，opID会突然从某个值变为1，而不是预想的一直单调加1

可能会出现多个clerk，此时不能通过单一的opID的增减，来判断是否有重复值，所以增加了ClientID这个变量

* opID不需要每次都加1，随机生成一个值即可

由于在clerk端通过锁，处理完一个op才能处理下一个，只需要保存上次处理opID的值，即可判断是否有重复opID

* opID除了在Get/Append等函数需要判断外，还需要在executeOP()处判断，因为有可能clerk向leader发送了两次同样的op
* 存在竞争问题：通过引入"github.com/sasha-s/go-deadlock"，发现在Put函数中，执行了kv.Lock()后，会卡在kv.rf.GetState()位置

因为在rf.GetState()中需要rf锁，而rf此时正在rf.applyCh <-位置，需要kv锁，互相等待。

* 需要在Get()/Put()等的无限循环中周期性释放锁，便于lastAppliedOpID[clientID]的更新，以及检查rf（kv.rf.GetState()）
* 注意一定要在return前面加上rf.mu.Unlock()来释放锁！
* 使用外部库来检测死锁。

## Lab3B: log compaction

### 过程

KVServer检测到maxraftstate小于 persister.RaftStateSize()，生成snapshot  ---> 将snapshot传输给raft，raft根据情况删除logEntries ，并调用persister.SaveStateAndSnapshot()保存snapshot。

### 需要实现的代码

**KVServer部分**

* 在每次执行完applyChan时，检查raft的大小
* 如果需要saveSnapshot，则生成data(encode之后的bytes类型)（包括了kv表以及client[lastAppliedOpID]），以及进行快照的logIndex，然后调用raft.SavePersistAndSnapshot()
* 实现ReadPersisit()，在KVServer启动时执行，传入的数据为kv.persister.ReadSnapShot，根据bytes类型的data，对kv表和lastAppliedOpID更新

**Raft部分**

* 当调用raft.SavePersistAndShnapshot()时，需要获得当前raft的state(bytes)，以及snapshot，然后调用persister.SaveStateAndSnapShot()
* 修改persist(),持久化的state得加上lastSnapshotIndex和lastSnapshotTerm
* 修改raft struct，加上lastSnapshotIndex和lastSnapshotTerm
* 修改appendEntries，需要判断args.lastLogIndex和当前raft的lastSnapShotIdnex，如果小于，return false，
* 修改appendEntries的args.lastLogIndex，不能通过logEntries长度直接获得，得加上lastSnapshotIndex
* 修改appendEntries的返回，reply.nextIndex可能会小于lastSnapshotIndex，此时调用InstallSnapShot
* 完成installSnapShot的RPC
* installSnapShot的时候，不会使logEntries长度为0，从而**总是可以通过访问最后一个元素**的Index和term，得到lastLogIndex lastLogTerm

1）修改struct和persisit()和readPersist()

2）~~添加getLastLogIndexAndTerm()，获得真实的lastIndex和term~~

3）

