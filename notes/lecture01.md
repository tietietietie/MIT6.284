# Introduction

## Paper:  "MapReduce: SimpliﬁedDataProcessingonLargeClusters"

### **概述：**

使用Map和reduce函数，对数据进行拆分和合并。（使用中间值键值对在两函数间传递数据）

![image-20200606084942827](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20200606084942827.png)

### **执行流程：**

![image-20200606085135161](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20200606085135161.png)

1，将文件分割成M份（区域性优化，worker1会优先在存储着split1的主机上运行，一般为16~64MB一份）

2，分为master进程和woker进程，master进程控制worker执行map程序，也控制着worker执行reduce函数（任务分配优化：均衡性能）

3，woker主机执行map函数，产生**中间值键值对**并存储在其内存

4，内存中的中间值键值对，周期性的往本地磁盘存储（减少网络带宽），并分割成R份（分割优化）。

5，当master通知worker执行reduce函数时，往对应的本地磁盘远程读数据，(中间值键值对进行排序，对相同的key进行聚合)。

6，对每一个key，执行reduce函数，并写到output file（最终只有一个，利用文件系统保证？）

7，所有map和reduce函数结束，通知user程序执行完成（最后总有一些速度很慢的map worker or reduce woker，利用备份进程进行优化）

## Lecture01：Introduction

分布式的问题：

* 计算机的并发/交互困难
* 局部失效
* 获得所需性能比较困难

Lab4：将数据分片，然后并发执行？？

Lab的debug非常困难，尽早开始。

存储（，通讯（tool)，计算：这些基础架构往往需要分布式。 

Dream：提供一个像非分布式的接口， 但底层是一个分布式的架构。

RPC:

Threads:

concurrency control:

1，可扩展性！！

单个存储服务器的性能很容易被消耗。

2，容错

如果有一千台主机，那几乎每天都会出现主机故障（或者网络故障）。

所以分布式系统的容错性能是非常需要的。

Availability：当发生了一些错误后，系统还是能够继续提供服务（有针对一些故障的容错机制）

Recoverbility：发生故障能够恢复。

两个工具：非易失存储（但是很难更新，恢复也很慢）复制（但是很难保持副本同步？）

解决方式：冗余主机？

3.一致性

例如分布式存储中，实现key/value的put和get，但是Key/value的表会有比较多的副本（由于容错机制），需要保证他们的一致性。

强一致性：保证get操作一定是最新的Put所更新的值（性能差，必须检查此次Get的值是最新）

弱一致性：get操作可能会读取到以前的Put操作（未被更新的值）

consistency/performance spectrum

由于容错性的要求，人们尽可能让不同的副本距离较远，从而造成Put操作的更新会很耗时。

如何保持弱一致性+高性能

MapReduce: Job = MapTasks + Reduce Tasks

可以使用流来优化reduce，而不是必须等待全部数据收集完成，才执行reduce函数

reduce的输出文件也是放在GFS上（也会消耗带宽）

## Lab1:MapReduce

### mcsequential.go

* 定义ByKey用来排序“中间键值对”
* 定义loadPlugin()函数来读取os.Args[1]中的map函数和reduce函数
* 程序执行流程
  * 打开文件，将文件名和文件内容（都是字符串），传入map函数中，产生kva键值对，将每个文件产生的kva都添加到中间键值对中。（类似于"apple" : "1", "banana" : 1, "apple" : "1")
  * 将中间键值对按照ByKey排序，并指定一个output文件。os.Create(oname)
  * 将排序后的中间键值对去重，得到每一个key和对应的values，类似于("apple" : "1", "1")，并传入reduce函数中，reduce函数统计每个key出现的次数，输出output（“2”）。
  * 最后程序将（key, output）写入文件。

### mrmaster

* 传入master所需要的文件（多个）以及reduce个数

```go
m := mr.MakeMaster(os.Args[1:], 10)
```

* 利用master的Done函数，每隔一秒检查worker是否全部完成

```go
for m.Done() == false {
    time.Sleep(time.Second)
}
```

### mrworker

* 传入map函数和reduce函数，创建worker

```go
mapf, reducef := loadPlugin(os.Args[1])
mr.Worker(mapf, reducef)
```

### master

* 定义Master结构体（my code）
* 定义RPC handler来处理worker的请求
* 定义一个server，启动net.listen，注册rpc服务，来处理所有rpc请求
* 定义一个makeMaster，用来创建master进程（my code)

### rpc

* 定义rpc变量，注意要全部大写开头(my code)

### worker

* KeyValue结构体，存放key 和 value 都是 string
* ihash:根据key，产生一个int，用来指派reduce
* 定义Worker函数，传入mapf和reducef两个函数，(mycode)
* 定义call函数，可以向master通过rpc发送信息

## lecture 2: infra : RPC and threads

### Go语言

为什么使用Go：

* 对线程，锁，同步，RPC的使用很容易
* 垃圾回收语言，内存自动回收，多线程的内存回收很重要，因为共享对象的回收需要所有线程使用完成
* 简单

### 线程

I/O并发：统一时间段，如果某一个I/O活动正在等待，则可以允许其他I/O活动

 并行：如果启动多个GoRoutine，多核处理器能够同时处理

方便：比如周期性的检查

如果没有并发，可以用事件驱动编程？？（通常用一个循环）（有一个事件表）（这是线性的，写起来很麻烦）（无法使用多核）

线程的数据交互比进程容易的多

挑战：

共享的数据容易出错：竞争  解决办法：加锁 （锁和变量是没有任何关系的。）

协作：使用channel， sync.Cond, waitGroup

死锁：互相等待对方解锁

### Web爬虫



