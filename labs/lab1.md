# lab1

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

### 数据结构定义

#### Task

包括：taskType, filaName, taskId（非常重要，在执行map任务时，取值为(0~n-1)能帮助我们确定中间键文件名称，在执行reduce任务时（取值0~m-1)，能帮助我们找到所需的中间值文件），mapCount， reduceCount。

#### Worker

申请任务 ---> 判断任务类型 ---> 执行对应任务 --- > 通知master执行的状态  ---> 申请任务

