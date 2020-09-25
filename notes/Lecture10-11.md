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

# Lecture11
## Paper:Frangipani: A Scalable Distributed File System
* 什么是Frangipani：分布式文件系统，运行在虚拟磁盘（远程）(分布式存储系统)Petal之上，可用性可扩展性较好。
* 结构：两层，底层是Petal，上层是运行着Frangipani文件系统的server
* ### 问题引入

**Frangipani server读写文件的策略**：使用缓存，每个workstation（WS）都有本地的缓存，且写回策略使write-back，即需要时写回。

**多WS读写文件存在的问题**

* WS1在缓存中修改或新增了一项文件grade entry，但是WS2在write back前读取不到这个文件。（cache coherency)
* 原子性文件，WS1和WS2同时修改或者在某一目录下新增路径，可能会造成冲突（相互覆盖）（atomicity)
* WS1在write back文件时crash了，如何保证别的WS不会看到不完整或者不合理的数据（crash recovery)

### 解决办法

**cache coherency**:使用分布式锁服务，提供文件全局管理（lock server ls)，原则是cache data前，必须在ls获得对应文件的锁，从petal读取相应数据，也需要获得锁，释放锁之前，需要write-back dirty page。流程如下：

1.WS1需要cache文件a，向ls发送锁请求，想获得对应锁

2.ls如果发现这么锁没有被占用（或者被占用但是***空闲（idle)***)，ls把锁分配给WS1

3.WS1获得锁后，把文件a缓存，可对其进行修改，暂时不需要释放锁，也不许write-back

4.WS2想要读写文件a，向ls发送锁请求

5.LS查看对应的锁是否被占用并且其状态是idle，如果lock处于这个状态，则LS要求WS1释放锁（revoke）

6.WS1先write-back，然后实放锁

7.WS2获得锁，也可获得更新后的数据。

**Atomoicity**:进行一项事务前，必须先获得该事务涉及资源的**所有锁**。

**Crash Recovery**:类似于WAL，本地的WS中会存在日志条目，记录每次修改metadata的log，只有Log传输到Patal后，才会修改真正的metaData。这样，如果WS在持有锁的时候crash了，Petal（或者另一个想要获得这个锁的WS），会执行log来恢复crashed WS的数据。

### 其他

* Frangipani是强一致（strong consistency)且用户无感知的（运行在**kernel层**）
* Frangipani是没有很复杂的admin level，每个WS都可以修改文件？
* Log有版本号，每次实放锁，版本号+1， 只有当版本号和log相同，说明上一次获得锁的WS crash了，此时replay log是有效的
* WS的高速缓存中，会出现日志填满的情况，此时需要把日志发送给Petal？
* 可以在不停机的情况下备份？？？why
  
## video：

Petal:virtual disk.

设计目的：小型的组织的共享文件系统，各用户对其他人操作可见，且组织互相信任。

为什么速度快：用户的大多数操作只针对自己的文件，其他人不需要看见，不需要经常write-back

为什么可扩展：文件系统的**去中心化**，没有一个central server，每个WS都有一个相同的Frangipani Server

Lock server：保存有file name < --- > owner的lock表项，同时WS中的Frangipani server中也保存着filename <---> lock(state) <---> content表项，其中state有两种状态：busy/idle，如果对应content没有被system call，则是空闲的（但是也不会被LS收走lock)

Rules: 1,WS中cache的每一项data，都必须有lock. 2,先得到锁，才能从Petal读数据。 3，先写数据到Petal，才能释放锁。

![image-20200924202725753](Lecture10-11.assets/image-20200924202725753.png)

如果WS存储了很重要的数据，但是没有任何人想要读它，它就会一直存储在WS的cache中吗？不会，WS会周期性的write-back。

Crash Recovery:如果持有锁的Server crashed，那么可能会造成Petal有不完整的数据，此时不应该立刻释放锁（不然其他的WS会读到这些数据），而是尽可能的恢复成完整数据。Frangipani使用WAL策略。WS的app试图修改数据，Frangipani server会先把log发送给Petal，然后进行修改，从而保证了app看到了修改后的数据时，log一定时存放在server上。

Frangipani log的特殊之处：

1，每一个WS都有自己的log（不像普通的tranzaction system，单一的一条Log）

2，每个WS的log不存在WS本地，而是放在Petal的WS专属区域。

Log组成：LSN：log序号号。array：petal number/ version number/ data(只涉及metaData，就是有关目录的data，而不涉及文件内容)

revoke操作流程：必须先write log（整个log)。然后write-back dirty page，然后release lock。注意log发送成功后，才会更新log

怎么检测WS crashed？必须有另一个WS需要一个锁，使LS调用invoke函数超时，确定WS crashed，这个需要锁的WS会去执行相应的log。

如果WS只写入了部分log就crash了？会reply一些已经完整的log，而丢弃未完成的log(整件事情的目的就是不能有不完整的数据！)

在Recovery（Replay)的过程中，如何保证不会对过时的数据进行replay？如下图：

![image-20200925144336351](Lecture10-11.assets/image-20200925144336351.png)



解决办法：每个file system的matadata都会有version number，每个Log涉及的data也会有一个version number。每次WS修改数据并写入log后，metadata中的VN会加一。所以当要执行log操作时，log中的VN如果小于或者等于patal中的VN，则不会执行。（只会执行 op's VN > petal block VN的Log)。

Recovery的过程中，需要锁吗？不需要，因为修改一个data时，可以保证这个data在WS crash后，没有被修改过（不然VN一定会变大，导致log不会执行）

## 参考
https://blog.csdn.net/plm199513100/article/details/108310623
https://www.jianshu.com/p/b4cbfca809ae
