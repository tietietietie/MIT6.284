# Lecture16

## Paper:Scaling Memcache At Facebook

### 什么是memcache?

分布式高速缓存系统，把原本数据库中的数据保存一部分在服务器的缓存中，提高访问速度。具体来说，是一个大的全内存hash表。

### 使用memcache进行读写

1，读 :使用Get(k)操作，如果memcache没有，则从database中select，并在memcache中set(k, v)

2，写：首先在database中updata(k,v)，然后memcache中delete(k)（防止看到过期数据）(如果先delete数据,在send完成之前另外的client读取key,此时memcache中保存的移然是过期数据,此时更新此key的用户,会发现自己的更新好像没有发生)

可以看出，可以使用memcache减轻database的负担。

## Video

### 用户不断增加时，服务器架构如何改变？

少量用户时：单个服务器上同时运行php处理代码和mysql

用户增多时：运行php占用了大量cpu，可以将php业务单独为front-end服务器，而mysql单独一台服务器。

用户继续增多时：mysql进行分片（sharding），分成不同的区域（设计到了分布式2pc提交等）

用户继续增多：不能把分片切成更小了？原因：1，出现单一key的hot spot。2，价格昂贵。所以facebook在FE服务器和DB之间增加了一层memcache。（memcache的读性能至少是DB的十倍）

### 问题

当get(k)未在memcache命中时，为什么memcache不主动向DB找数据，而是返回false，然后FE server向DB提出get(k)，再把未命中的（k,v）put到memcache? 

这样做看上去增加了通讯次数，但为了把memcache作为单独的组件独立于DB（look-aside cache)

由于memcache的命中率很高（99%）一旦有一个memcache失效，会造成DB的负载快速上升，从而系统崩溃。

### ＦＡＣＥＢＯＯＫ架构

用户要求：很多情况下，用户并不要求非常精确的一致性，他们对于读到的过时一两秒的数据,不会有察觉. 同时如果用户刚刚对数据进行了修改,此时还是看到过期数据的话,是不能接受的.

**Region**:分为master/slave,其中所有写操作都要发送到master的DB

### 为什么使用delete()来处理更新后的数据(而不是set)

如果使用set,可能会使memcache中的数据是错误的.

C1: x = 1  --> DB

C2: x = 2  --> DB

C2: set(x, 2)  --> memcache

C1: set(x, 1)  --> memcache

### 使用额外硬件提升性能

1,分区: 将数据分散到多个服务器:优点是内存更多,内存方面效率更高,缺点是无法解决hot key, 同时和多个server通讯.

2,复制: 将数据复制成多分放在多个服务器: 优点:可以解决hot keys, 也可以减少TCP通讯,但是内存的使用率低,存储的数据少.

### Region结构

分为多个cluster和一个DB层,cluster中有多台FE server和多台memcache server.

为什么一个region里面要分成多个cluster呢?  减少tcp次数, 因为tcp次数为O(n^2),n为FE和memcache数量,n过大会导致丢包?

增加一个新的cluster会造成DB突然负载增大,解决方法为冷启动, 冷启动是新增加的FE发生get miss时,会去相邻cluster的memcache server中找data,而不是直接去DB取.

### Thurdering Herd(雷霆羊群效应)

假设有1000个FE对一个memcache不断进行get请求,此时一个FE修改了该key, 从而删除了该key, 1000个服务器同时向DB查询该key,造成拥堵,memcache从而收到了1000此put(key, v)操作,就算这些值是一样的.

解决办法:第一个发送get并Miss的FE sever会从memcache得到该key的一个Lease, 其余FE则通知等待(10ms后再发送?), 当第一个FE查询到了新的(key,value)后, memcache删除lease并更新数据, 从而999个FE都可以得到数据而不用查询DB了.