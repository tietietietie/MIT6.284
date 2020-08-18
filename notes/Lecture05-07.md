# Lecture05 Go Thraeds and Raft

## Preparation: The Go Memory Model

* 为了保证当前Goroutine能观察到其余routine的操作结果，使用lock()或者channel保证同步
* channel能够保证send操作发生在receive之前。

```go 
var c = make(chan int, 10)
var a string
fun f() {
    a = "hello world"
    c <- 0
    // close(c)同样效果，因为close一定是发生在recieve之前
}

func main() {
    go f()
    <- c
    print(a)
}
```

* unbufferd的channel，recieved一定是发生在send完成之前（理所当然，不能会造成数据丢失），同理，第k此recivied操作一定发生在k+C次send操作前，C为channel容量，容量能够保证并发数量。
* Lock()也可以保证顺序，当前lock()一定发生在上次Unlock()之后。
* 编译器会调整赋值或读写的顺序，如以下程序，最终打印结果可能是2 0

```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

* goroutine之间很可能观察不到各自的变量变化（就算是全局变量）,如下，main可能进入死循环，就算观察到了g != nil，也可能输出空串

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

## Video

Go语言处理多线程问题处理

1,将变量显示的传给go func，而不是直接引用外部变量，因为无法确定goroutine执行时的外部变量是什么值

waitGroup的作用，等待一定数量的线程执行完成

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(x int) {
            sendRPC(x)
            wg.Done()
        }(i)
        wg.Wait()
    }
}

func sendRPC(i int) {
    println(i)
}
```

2,使用done来终止另一个goroutine（必须使用lock或者channel，来保证共享变量能被观察到）

```go
package main

import "time"
import "sync"

var done bool
var mu sync.Mutex

func main() {
	time.Sleep(1 * time.Second)
	println("started")
	go periodic()
	time.Sleep(5 * time.Second) // wait for a while so we can observe what ticker does
	mu.Lock()
	done = true
	mu.Unlock()
	println("cancelled")
	time.Sleep(3 * time.Second) // observe no output
}

func periodic() {
	for {
		println("tick")
		time.Sleep(1 * time.Second)
		mu.Lock()
		if done {
			return
		}
		mu.Unlock()
	}
}
```

3.Lock可以保证一个**区域**是原子性的

4，Busylocking

```go
for {
    mu.Lock()
    if count >= 5 || finished == 10 {
        break
    }
    mu.Unlock()
    time.Sleep(50 * time.Millisecond)
}
```

use condition lock: 在cond.Wait()处，当前线程会释放锁，进入等待队列，只有某个线程出发cond.Broadcast()后，线程才会重新得到锁。一般cond的语句在lock和unlock之间，保证是确实有锁的

```go
	for i := 0; i < 10; i++ {
		go func() {
			vote := requestVote()
			mu.Lock()
			defer mu.Unlock()
			if vote {
				count++
			}
			finished++
			cond.Broadcast()
		}()
	}

	mu.Lock()
	for count < 5 && finished != 10 {
		cond.Wait()
	}
	if count >= 5 {
		println("received 5+ votes!")
	} else {
		println("lost")
	}
	mu.Unlock()
}
```

 5.channel是阻塞形式的,send等待receive，receive等待send，他们会同时发生（unbufferd)

```go
func main() {
	c := make(chan bool)
	c <- true
	<-c
}
```
# Lecture06-07 Fault Tolerance - Raft
## Preperation(paper)
### Log复制
* 一次顺利的log复制流程
leader收到client请求 ---> leader在自己的log上添加一项新entry ---> 向follower发送AppendEntry ---> entry被成功复制到大部分的follower ---> leader执行这项entry，并将结果返回给client
* Entry = 状态机一次操作 + term + index
* 能够被安全执行的entry称为commited, leader会保存最高commited entry索引并发送给followers，这样其余server就知道哪些entry是可以执行的了。
* Entry性质：term + index相同的两个entry，他们的内容是一样的,并且他们之前的entry都是一样的。如何保证：leader在某一term下的某一log index，最多创建一条entry，并且在发送AppendEntries时，会检查follower的最近index + term是否与自己相同。
* 如何处理不一致的follower：leader会强制follower复制自己的log（不一样的log会被改写），leader会找到follower的最新的一致节点，把该点后面的entry都删除，然后把自己的log复制上去。（如何找到点：nextIndex[]中数字不断减一)
* leader永远不会删除或者修改自己的log entry！！
### 安全性
上述log复制策略并不能保证状态一致性，一种可能的情况：失联follower成为leader，会对之前term的commited log进行覆盖。如何保证新当选的leader保存有所有的commited entries呢？
* 选举限制：candidate的log在其他server中是最新的？
* 提交之前term的log：在新的term中，不会利用count副本来提交旧log，只有当前term的log会通过计数的方式提交。
* 安全性证明(反证法)：假设当前leader U中，不存在old term log(term T) ---> 必存在一个server，同时接收了这个缺失的log，并且投票给了U ---> voter在投票的时候，一定存储着这个log ---> voter和leader U的log必须是up-to-date的 ---> 此时造成冲突
* Follower和candidate的失效处理方式：不断的发送request，直到有响应为止。
### Log压缩
压缩办法：快照
快照内容：最后inclued log的term和index,以及状态机的状态
快照方法：每个server都可以独立的创建自己的快照，如果follower落后太多，leader也会发送InstallSnapshot请求，来修改follower的log
## Video
在repication system中，可能会有一个单节点，在做重要决策，此时有必要防止单点故障。
**脑裂**：client不会同时请求两个server ---> client请求多个replicated server的一个 ---> 网络故障，两个client分别修改不同server的值，而值没有同步 ---> 脑裂
如何避免partition ---> majority vote， 2 * f + 1 servers can suffer f failure
另一个重要特征：在两次leader选期中的majority中，至少有一台机器的term是重叠的。
### Raft
raft是一个库，供上层服务(eg k/v server)调用实现一致性。
log至关重要，为了保证所有状态机的执行顺序相同，执行操作相同。
log收到的操作可能会被丢弃，leader的log需要被重新发送到一些延迟的follower。而且log也会被写入磁盘，当server重启时执行。 
所有服务器失效后重启 ---> 选择出一个Leader ---> leader通过发送心跳（appendEntries）知道majority当前的日志提交情况 ---> 强制majority的提交情况相同 ---> 所有服务器从头开始执行要leader commit point
start(command)(index, term ):准备将该请求发送给各个server，当start返回时，确认了majority的备机已经commit当前command，此时leader才会将该command加入到apply队列（start command ---> committed by majority ---> add to applyCh)
#### leader selection
* 为什么需要leader：能够容易达到一致性。
* 如果leader有单向网络故障（会发送心跳但是收不到client请求），则raft系统会崩溃
* 随机化选举周期，并且需要每次开始选举后，重新选取随机值。
#### log append
* 可能存在leader发送appendEntries后commit或者没有commit，但是raft始终把他当成commit处理？
## Video
* leader如何保证follower能够和他的log相同？
* leader的选择更加谨慎
* Fast backup
* 根据persist的要求，你必须先把persist的内容如Log写到硬盘，然后再发送rpc