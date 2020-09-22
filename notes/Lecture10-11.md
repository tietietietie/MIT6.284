# Lecture 10

## Papar: Amazon Aurora + Video

### 亚马逊云服务的发展背景

* EC2：对服务器进行抽象（VMM），向上是各种操作系统等，提供可扩展的服务（如web server等），但是多个web server只有一个DB server。

* EBS：主从数据库服务器，使用chain replication策略，提高系统的容错性。缺点：1，复制时NetWork开销大，因此两个服务器通常在一个data center。2，同一个data center容错率低。3，chain repliaction造成的延迟问题。

* RDS：类似于fig2的结构，主要特点是，把repliaction转移到了两个Availability Zone(AZ)(可以理解为data center)。

  ![image-20200922211032771](Lecture10-11.assets/image-20200922211032771.png)

* 上述结构的缺点：大量Network，一些操作必须串行，如1,3,5。

### 数据库的简单介绍

* 理解两个概念：事务/crash恢复
* 事务：原子化的一系列操作基合，要么成功，要么失败。一般的实现方式中，需要使用锁，对transaction中的所有数据进行锁定。

* 数据库的一般模型为：上层的DB 软件和底层的存储系统。DB软件着，拥有底层存储系统的data page缓存，用户对数据库发送命令进程操作更新（transaction)后，DB修改了cache中的值，得到了更新的log，只要当更新的log被添加到了存储系统的write head log（WAL）中，才算此次transaction完成。
* log中存储着变量的old value和updated value，为什么需要old value呢？因为有些数据库系统中，logs不是等一个transaction全部完成后，再添加到WAL的，而是生成一条，添加一条，此时如果这次transaction失败，则需要undo操作，会用到old value。(保证原子性)
* redo操作是指，数据库崩溃后，还能根据log进行redo，从而恢复。

### Aurora

![image-20200922212413254](Lecture10-11.assets/image-20200922212413254.png)

基本思想：把log剥离出来，放在了storage系统中。

两个亮点：1，6个replica的Quorum机制。降低由于slow server带来的延迟问题。2，虽然副本变多，但是Network的压力变小了，因为只需要传输log，不需要像以前一样传输data page（远比Log大）

Quorum机制：设有N个副本，写操作要在W个副本上成功，读操作要在R个副本上成功，此时W + R >= N+1，就能保证每次读到的都是最新数据。（可以自己根据需要定义W,R，满足条件即可）

写数据：只在Primary DB（唯一一个）上进行写，且不需要每次都更新data page，采用old page[i] + 相应log entries的形式。当然需要把log都发送到W个storage servers上(Quorum读），才返回成功。

读数据：可以有多个read-only DB进行读，提高吞吐量。通常情况下，DB可正常运行，它会直到这六个storage servers上，log的最新ID，从而就可以直到哪些个storage server是有最新数据的，此时它只需要向该server请求读取data page[i]，即可得到数据（当然那个storage server得更新page了）。特殊情况即DB崩溃（不是storage server崩溃，这个是能容错的）时，需要进行Quorum读，保证能够恢复到最新的数据。（恢复时的小细节：由于DB崩溃时，很有可能是在事务处理的过程之中，需要找到最新的uncommi transaction，进行undo操作，保证原子性）

存数据：使用Protection group（PG）存储分片（segmentation),分片是必要的，为了存储海量数据，也为了提高吞吐量。常见PG形式：把一个数据库每份10G大小的切片，每个切点有6个副本，放在6个不同的服务器上，组成一个PG。Log怎么存放呢，log中修改的数据是知道在哪个PG的，发送到对应的PG即可。

一个存储服务器失效，怎么把里面成百上千个PG切片恢复呢，如有100个不同的PG分片丢失了，并发的在新的100个服务器来复制即可。

### 心得

* 一般设计数据库时，数据库系统和存储系统是分开的，为了低耦合，为了通用的存储系统，但在这篇论文中，模糊了上层系统和底层存储的界限，使性能大幅度提高了
* 云基础架构需要考虑的一些问题：整个AZ失效，出现短暂的慢副本（导致同步变慢），网络成为了瓶颈，所以降低了传输的数据量（只传log），但是相应增加了CPU计算量（需要计算6次），但显然在亚马逊看来，network负载比cpu计算量重要的多。



