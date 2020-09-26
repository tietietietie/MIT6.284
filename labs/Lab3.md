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

**Clerk Struct**: 包括所有K/V server的地址，以及最新处理完成的opID，以及上次RPC时连接的leaderID，以及锁（保证opID的原子性更新）

**GetArgs**:除了Key之外，还需要唯一的opID。

**Get**:第一次尝试连接的serverID初始化为上次RPC成功的leaderID，RPC的timeout时间设置为40ms，根据reply.Err判断是否操作成功。

**AppendPut RPC代码类似**

**KVServer Struct**:定义一个k/value表格（map)

