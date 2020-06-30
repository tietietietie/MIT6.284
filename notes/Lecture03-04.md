# Lecture03-04

## Paper:GFS

### 绪论

#### GFS与其他分布式文件系统的不同

* 组件失效被认为是正常的（常态）
* 单个文件很大（GB级别），从新设计I/O操作
* 文件以添加新文件的方式进行改变，而不是直接改写。
* GFS能够为许多应用提供便利。

### 设计

#### 假设/前提

* 持续检测错误，发生错误能迅速恢复
* 支持小文件，但是以大文件为主（> 100M，通常GB）
* 大流量的读取和小流量的随机读取（存储），对性能的保证通常是大流量读取
* 对多用户同时改变文件（添加）有明确的语义定义，尽可能减少同步带来的损耗
* 注重持续高带宽而不是低延迟

#### 接口

提供类似常见文件系统的操作接口：比如create, delete, open, close, read, write，特色操作：snapshot：快速复制文件或者目录树，record append：多用户同时写入文件（原子性不需要额外锁）

#### 架构

![image-20200625134429699](Lecture03-04.assets/image-20200625134429699.png)

#### 单一Master

* 不会直接与Client进行数据传输，会通知Client应该与哪个chunkserver进行数据交换

#### Chunk Size

* 选择64MB，优点如下
  * 减少与master的交互，并能轻松的缓存chunk位置
  * 能够减少带宽消耗
  * 减少元数据的数据量，是我们能够把元数据保存在内存中
* 缺点
  * 可能会造成“热点”chunk server

#### 元数据

分为三类：文件名/chunk名称，file < --- >chunk映射，chunk及副本存放位置，前两者会有log保存在本地和远程，chunk存放位置不用保存，master启动时，chunk server会主动发送。

* 操作日志：保存着元数据的修改信息，非常重要，保留多分，用于master复原，为了节省复原时间，设置了checkpoint，定期总结。
* 元数据修改前必须前保存日志。

### GFS集群操作流程

#### Master NameSpace管理

* 对每一个文件和目录都分配了一个读写锁，从而支持对不同的namespace并发操作
* 对于“/d1/d2/.../dn/leaf”这个路径，首先会获得“/d1/d2/.../dn”父路径的读锁

#### 读取文件

如上图所示，分为四步

* 根据文件名和读取偏移量，得到chunk index，发送给master
* master向客户端返回chunk的Handle以及所有副本的位置，客户端以文件名和chunk index作为健进行缓存
* 客户端根据chunk handle和所需读取的范围，向其中一份副本所在的chunk server发送请求

* 最终该chunk server把数据传给客户端

#### 写入文件

具体流程见下图

1. 客户端向 Master 询问目前哪个 Chunk Server 持有该 Chunk 的 Lease
2. Master 向客户端返回 Primary 和其他 Replica 的位置
3. 客户端将数据推送到所有的 Replica 上。Chunk Server 会把这些数据保存在缓冲区中，等待使用
4. 待所有 Replica 都接收到数据后，客户端发送写请求给 Primary。Primary 为来自各个客户端的修改操作安排连续的执行序列号，并按顺序地应用于其本地存储的数据
5. Primary 将写请求转发给其他 Secondary Replica，Replica 们按照相同的顺序应用这些修改
6. Secondary Replica 响应 Primary，示意自己已经完成操作
7. Primary 响应客户端，并返回该过程中发生的错误（若有）

#### 副本管理

创建副本可能是如下三个时间：创建chunk，为chunk重新备份，副本均衡。

#### 删除文件

采用延迟删除策略，首先namespace中的文件不会马上移除，进行重命名即可，经过一定时间后在进行删除。

NameSpace文件被删除后，其对应的chunk的引用计数减1，当某一chunk的引用为0，则master将该chunk的元数据在内存中全部删除。chunk server根据心跳通信，汇报自己的chunk副本，如果这个chunk不存在，chunk server**自行移除**对应的副本

### 高可用机制

#### Master

* 以先写日志形式，对集群元数据进行持久化，日志被确实写出前，master不会对客户端的请求进行响应，日志会在本地和远端备份。
* 使用检查点来提高master恢复速度
* Shadow Master：提供只读功能，同步Master信息，分担Master读操作压力。

#### Chunk Server

* 每一个chunk都有对应的版本号，当chunk server上的chunk的版本号与master不一致时，便会认为这个版本号不存在，这个过期的副本会在下次副本回收过程中被移除
* 同时用户读取数据时，也会对版本号进行校验

#### 数据完整性

* 利用校验和（32位）检查校验和块（64KB）是否有损坏
* 校验和保存在chunk server内存中，每次读操作时进行校验，每次写操作进行重新计算
* chunk server在空闲时会周期性的扫描并不活跃的副本数据，确保有损坏的数据能被及时检测到

### FAQ

* GFS牺牲了正确性（可能会有过时数据/重复数据等），是为了保证系统的性能和简洁性，总之，弱一致性系统通常较简洁，而保证强一致性，则需要更加复杂的协议。

## Video：[深入浅出Google File System](https://www.youtube.com/watch?v=WLad7CCexo8)

### 如何保存文件

#### 保存大文件（单机）：

![image-20200629165005707](Lecture03-04.assets/image-20200629165005707.png)

#### 保存超大文件（分布式）

**如何降低master中的元数据信息，以及减少master的通讯**：

master不会保存磁盘的偏移量，只需要保存chunk所对应的chunkserver，在chunkserver中，再查找磁盘的偏移量即可。

### 数据损坏

#### 发现数据损坏

每个blcok中保存一个checksum，进行校验。每次读取都进行hash校验

#### 减少数据损坏的损失

创建副本（三个）

如何选择副本的chunkserver：硬盘使用率低，跨机架跨数据中心

#### 如何恢复损坏的chunk

chunkserver想master求助，找到副本的位置

### Chunkserver挂掉

#### 如何发现

心跳，周期性的向master发送信息。

#### 如何恢复

master启动修复进程，根据剩余chunk的多少进行优先级排序。chunk副本越少，越先进行修复

### 热点

热点平衡进程：记录访问频率，带宽等，如果某个server访问过多，创建多个副本server

### 如何读文件

![image-20200629170225216](Lecture03-04.assets/image-20200629170225216.png)

### 如何写文件

注意primary是写的时候临时指定的。

注意先缓存再写入硬盘（防止出错）

![image-20200629170616555](Lecture03-04.assets/image-20200629170616555.png)
