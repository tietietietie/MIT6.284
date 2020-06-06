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