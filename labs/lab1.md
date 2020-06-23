# lab1

实验环境：WSL:Ubuntu 18.04  + VsCode

## 1，理解题意

阅读提供的代码，理解功能

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
* 定义Done函数，表明job完成了

### rpc

* 定义rpc变量，注意要全部大写开头(my code)

### worker

* KeyValue结构体，存放key 和 value 都是 string
* ihash:根据key，产生一个int，用来指派reduce
* 定义Worker函数，传入mapf和reducef两个函数，(mycode)
* 定义call函数，可以向master通过rpc发送信息

### a few rules

* 注意输出文件格式

### 步骤

* 定义worker的call函数，向master请求一个任务。
* 定义master的响应函数，将未处理的文件交给worker处理。
* 定义中间键名称 mr-X-Y,其中X为Map task number, Y为reduce task number
* 可以利用Go的json包，把中间键值对存储为json格式文件
* 使用iHash()函数，帮助选择对应的reduce wiker
* master是并发的，注意锁定共享变量。
* 每10秒检测一次worker是否完成，如果没有完成，则放弃这个worker。

## 2，配置实验环境

实验环境：WSL（ubuntu 18.04) + VSCode

安装Go 1.13，并解压到/usr/local路径

```
sudo tar -zxvf go1.13.6.linux-amd64.tar.gz -C /usr/local
```

添加环境变量，参考[这里](https://www.jianshu.com/p/c43ebab25484)

安装GIt，报错：

>Could not get lock /var/lib/dpkg/lock-frontend - open (11: Resource temporarly unavailable)

解决办法：

```
sudo rm /var/lib/dpkg/lock
sudo rm /var/lib/dpkg/lock-frontend
```

注意：不要安装VSCode的插件，因为无法下载，还会导致go tool运行出错。

## 3，MapReduce实现

### 数据结构定义/函数功能介绍/文件功能介绍

#### Task （Struct）

包括：taskType, filaName, taskId（非常重要，在执行map任务时，取值为(0~n-1)能帮助我们确定中间键文件名称，在执行reduce任务时（取值0~m-1)，能帮助我们找到所需的中间值文件），mapCount(执行reduceFunction要用）， reduceCount（执行mapFunction的时候要用）workerID(因为可能出现两个worker执行相同任务的情况，此时任务是否完成，应该以master中任务的workerID确定）。

#### TaskState (Struct)

包括：任务状态：入队？超时？错误？已完成？（0，1，2，3）

任务执行ID：workerID

任务开始执行时间

#### Worker.go

作用：申请/执行任务

worker结构体：包括mapF和reduceF, ID

申请WorkerID ---> 申请任务 ---> 判断任务类型 ---> 执行对应任务 --- > 通知master执行的状态  ---> 申请任务

#### Master.go

作用：调度/分配任务

1, 每个一段时间，检查所有任务的状态：包括判断所有任务已完成？将执行太久的任务/报错的任务重新放入准备任务队列

​	1）如果map任务全部完成，则启动reduce任务

​	2）如果reduce任务全部完成，则将Done参数设置为true

2，Master结构体：存放着所有需要共享的变量，包括：master当前管理的任务类型（map/reduce），所有任务的状态（入队？已完成？在执行？故障？）（用一个数组存储，index是TaskId(0~n-1, 0~m-1)， value是taskState，四种），Done：表示全部任务（map and reduce）已完成，files(输入文件)， nReduece(切片数量)， 互斥锁。

3，执行过程：启动所有map任务（入队） ---> 将管道中的任务分配给woker ---> 每隔一定时间检查所有任务状态，处理超时/失败任务，重新入队，如果已完成 ---> 启动所有的reduce任务 ---> 将管道中的任务分配给worker --->一定时间检查任务状态，如何任务全部完成  ---> Done = true

4，分配任务：收到任务请求（RPC) ---> 在管道中获取任务 ---> 处理任务（包括确定此任务workerID，改变任务状态，确定此任务的开始时间） --->  将任务分配给worker

5，标记任务完成：收到任务完成请求 ---> 判断当前m.taskPhase和args.taskPhase是否对应，workerId是否对应（因为可能有非常慢的worker） ---> 标记任务状态为已完成

### 代码

#### 初始化map任务

将Master结构初始化，确定files和nReduce，并且开启Map任务

#### 请求任务

```go
func (w *worker) requestTask() Task {
	args := RequestTaskArgs{}
	args.WorkerID = w.workerID
	reply := RequestTaskReply{}
	if ok := call("Master.AssignOneTask", &args, &reply); !ok {
		log.Println("worker %d cannot request a task", w.workerID)
		os.Exit(1)
	}
	return *reply.Task
}
```

worker在请求任务中，需要判断ok的值，不然worker永远无法退出？或者连接超时？反正有Bug，无法通过。

#### 注册Worker

```go
func (w *worker) register() {
	args := RegisterArgs{}
	reply := RegisterReply{}
	call("Master.RegisterWorker", &args, &reply)
	w.workerID = reply.WorkerID
}
```

#### 报告任务完成

args: 任务类型， 任务ID， 是否完成， workerID

将master的任务标记为已完成，需要满足：任务类型和当前Master处理的任务类型一致，workerID与master中的任务状态的workerID一致，并且报告的是已完成

#### 执行map任务

mapf(args1, args2) : args1 : fileName, args2 : content， res : key-value数组

将key/value数组分成nReduce份，分别存储在mr-X-Y名称的文件中

#### 执行reduce任务

reduce()

读取所有的中间键值对文件，全部存入一个地方（kva） --->  将相同的key聚合在一起（map)（key, values) ---> 对每个key进行reduce操作

将reduece结果整理到res中，并统一写入到mr-out-*文件中，一行行的写入是有问题的，不知道为啥

正确代码：

```go
func (w *worker) execReduceTask(task Task) {
	maps := make(map[string][]string)
	for idx := 0; idx < task.MapCount; idx++ {
		fileName := fmt.Sprintf("mr-%d-%d", idx, task.ID)
		file, err := os.Open(fileName)
		if err != nil {
			w.reportTask(task, false)
			return
		}
		dec := json.NewDecoder(file)
		for {
			var kv KeyValue
			if err := dec.Decode(&kv); err != nil {
				break
			}
			if _, ok := maps[kv.Key]; !ok {
				maps[kv.Key] = make([]string, 0, 100)
			}
			maps[kv.Key] = append(maps[kv.Key], kv.Value)
		}
	}
	//指定slice的长度和容量
	res := make([]string, 0, 100)
	for k, v := range maps {
		res = append(res, fmt.Sprintf("%v %v\n", k, w.reducef(k, v)))
	}

	if err := ioutil.WriteFile(fmt.Sprintf("mr-out-%d", task.ID), []byte(strings.Join(res, "")), 0666); err != nil {
		w.reportTask(task, false)
	}

	w.reportTask(task, true)
}
```

有BUG的代码

```java
kvs := make(map[string][]string)
for i := 0; i < task.MapCount; i++ {
	fileName := fmt.Sprintf("mr-%d-%d", i, task.ID)
	file, err := os.Open(fileName)
	if err != nil {
		log.Fatalf("cannot open reduceFile %v", fileName)
		w.reportTask(task, false)
		return
	}
	dec := json.NewDecoder(file)
	for {
		var kv KeyValue
		err := dec.Decode(&kv)
		if err != nil {
			break
		}
		//判断key是否存在于kvs
		if _, ok := kvs[kv.Key]; !ok {
			kvs[kv.Key] = make([]string, 0)
		}
		kvs[kv.Key] = append(kvs[kv.Key], kv.Value)
	}
}
//将reduce结果存入文件
fileName := fmt.Sprintf("mr-out-%d", task.ID)
file, err := os.Create(fileName)
if err != nil {
	log.Fatalf("cannot create %v", fileName)
	w.reportTask(task, false)
}
for key, values := range kvs {
	_, err := file.WriteString(fmt.Sprintf("%v %v/n", key, w.reducef(key, values)))
	if err != nil {
		log.Fatalf("cannot write string in %v", fileName)
		w.reportTask(task, false)
	}
}
w.reportTask(task, true)
```

### 测试结果

> sh test-mr.sh	

通过所有测试，但是会有：Unexpected EOF提示。

![image-20200623101852199](lab1.assets/image-20200623101852199.png)