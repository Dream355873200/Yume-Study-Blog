---
title: Experiment1：MapReduce
date: 2026-02-10
tags:
  - Multi-Threading
  - "#c-plus-plus"
  - MapReduce
---
# MapReduce 实验实现报告 (MIT 6.824 Lab 1)

## 1. 前言

**参考论文：** [[mapreduce.pdf]]

### 个人理解

嗯实际上，经过我粗略地阅读论文并观察 Lab1 提供的样板代码后，我发现 MapReduce 架构核心是一个主节点 **Master**（Lab 中称为 **Coordinator**）和多个 **Worker**。

**关于通信机制：** 与论文描述不同的是，我发现Lab 实验使用了 `net/rpc`。这种rpc实际是
**单向通信**，即只能由客户端（Worker）主动发往服务端（Coordinator）。因此，我实现的Lab 具体架构和算法实现与论文原型的“Master 主动推送”逻辑是有所区别的。

**关于超时控制：** Lab 要求的超时控制是通过 Master 定期检查任务状态实现的。在我的实现中，我采用了一种简单有效的策略：

1. Worker 完成任务后发送 `Complete` 消息。
2. Master 在分配任务的同时启动一个协程（Goroutine），睡眠 10 秒（实验要求）。
3. 睡眠结束后检查该任务是否被标记为“已完成”，如果没有，则将其重新放入待分配队列。

这份代码的架构是我综合了老师的样板代码和论文原理后的结果。虽然可能不是最完美的，但对我现在的水平来说，解决其中的逻辑冲突已经很有挑战性了，也很有启发。

---

## 2. MapReduce 具体架构

### Master 节点：状态机模式

我的 Master 节点采用**状态机模式**来管理整个 Job 的生命周期。通过 `RunningStage` 变量来控制当前处于 Map、Reduce 还是结束阶段。

#### 核心函数：TaskSend (RPC)

这是最重要的函数。Worker 会一直循环调用它来请求任务。Master 根据当前阶段判断该给 Worker 分配什么工作。

```go
  
func (c *Coordinator) TaskSend(args TaskArgs, reply *TaskReply) error {  
    c.lock.Lock()  
    defer c.lock.Unlock()  
  
    //Map  
  
    switch c.RunningStage {  
    case 1:  
       {  
          if len(c.InputFile) == 0 {  
             reply.TaskType = 0 // 等待任务.5second  
             return nil  
          }  
  
          reply.TaskId = c.NowID  
  
          c.NowID++  
  
          reply.TaskType = 1  
  
          reply.FileName = append(reply.FileName, c.InputFile[0])  
          c.InputFile = c.InputFile[1:]  
  
          c.Tasks[reply.TaskId] = &TaskStatus{0, reply.FileName, 0}  
          go c.MapTaskTest(reply.TaskId, reply.FileName) //Task检测  
       }  
    case 2: //Reduce  
       {  
          if len(c.ReduceNum) == 0 {  
             reply.TaskType = 0  
             return nil  
          }  
  
          reply.TaskId = c.NowID  
          c.NowID++  
          reply.TaskType = 2  
          reply.ReduceNum = c.ReduceNum[0]  
          c.ReduceNum = c.ReduceNum[1:]  
  
          c.Tasks[reply.TaskId] = &TaskStatus{0, nil, reply.ReduceNum}  
          go c.ReduceTaskTest(reply.TaskId, reply.ReduceNum)  
       }  
    case 3:  
       {  
          reply.TaskType = 3 //结束  
       }  
    }  
  
    return nil  
}


```

这个RPC服务函数是最重要的函数，
woker节点在发送InitCall获取nReduce后就一直循环调用TaskSend函数获取执行的任务，
让Master节点来判断什么时候该切换任务状态

### 状态维护：Coordinator 结构体

为了支撑上述逻辑，Master 需要维护全局的待处理队列和任务状态表：
```go
type Coordinator struct {  
    // Your definitions here.  
    lock         sync.Mutex  
    RunningStage int                 //1 map,2reduce,3over  
    InputFile    []string            //输入的文件名  
    ReduceNum    []int               //reduce从1-10  
    ToMapNum     int                 //待map的任务量  
    ToReduceNum  int                 //待Reduce的任务量  
    NowID        int                 //Task唯一自增ID  
    Tasks        map[int]*TaskStatus //Task  
  
    MapNum int //map的初始化编号  
  
    NReduce int //NReduce  
}
type TaskStatus struct {  
    Status    int //0未完成，1已完成,2超时  
    Files     []string  
    ReduceNum int  
}
```

我处理 Worker 错误采用了**队列重入**的思想。
维护一个InputFile和ReduceNum
如果 `MapTaskTest` 发现任务超时，就把对应的文件重新放回 `InputFile`和`ReduceNum` 队列。
让其他的Worker节点来重新读取和执行

## 3. Worker 节点实现

Worker 同样是一个状态机。它不断向 Master 请求任务，根据返回的 `TaskType` 执行 Map、Reduce 或者直接退出。

### Worker 主循环逻辑
```go
func Worker(sockname string, mapf func(string, string) []KeyValue,  
    reducef func(string, []string) string) {  
    coordSockName = sockname  
  
    err := InitCall()  
    if err != nil {  
       panic("Init fail")  
    }  
  
    for {  
       reply, err := TaskRequest()  
       if err != nil {  
  
       }  
  
       intermediate := []KeyValue{}  
       switch reply.TaskType {  
       case 0:  
          {  
             time.Sleep(5 * time.Second)  
          }  
       case 1:  
          {  
             for _, filename := range reply.FileName {  
                file, err := os.Open(filename)  
                if err != nil {  
                   log.Fatalf("cannot open %v", filename)  
                }  
                content, err := ioutil.ReadAll(file)  
                if err != nil {  
                   log.Fatalf("cannot read %v", filename)  
                }  
                kva := mapf(filename, string(content))  
                intermediate = append(intermediate, kva...)  
                file.Close()  
             }  
  
             buckets := make(map[int][]KeyValue)  
  
             for _, kv := range intermediate {  
                ReduceNum := ihash(kv.Key) % nReduce  
                buckets[ReduceNum] = append(buckets[ReduceNum], kv) //创建一个哈希桶,对应每个reduce  
             }  
  
             tempFiles := make([]*os.File, nReduce)  
             tempNames := make([]string, nReduce)  
             for i := 0; i < nReduce; i++ {  
                tempFile, err := os.CreateTemp(".", "mr-tmp-*")  
                if err != nil {  
                   log.Fatal(err)  
                }  
                tempFiles[i] = tempFile  
                tempNames[i] = tempFile.Name()  
  
                enc := json.NewEncoder(tempFile)  
                for _, kv := range buckets[i] {  
                   err := enc.Encode(&kv)  
                   if err != nil {  
                      log.Fatal(err)  
                   }  
                }  
  
                // 关闭文件  
                tempFile.Close()  
             } //根据每个桶创建一个临时文件  
  
             for i := 0; i < nReduce; i++ {  
                finalName := fmt.Sprintf("mr-%d-%d", reply.TaskId, i)  
                err := os.Rename(tempNames[i], finalName)  
                if err != nil {  
                   log.Fatal(err)  
                }  
             } //原子重命名每个临时文件  
  
          }  
       case 2:  
          { //TODO先请求要处理的reduceNum,再查找本地文件所有对应reduceNum的mapTask编号，再去请求查找所有标记为已完成的Task而不是超时的Task，根据已完成的task编号处理所有的对应文件，输出。  
  
             pattern := "mr-*-" + strconv.Itoa(reply.ReduceNum)  
  
             files, err := filepath.Glob(pattern)  
             if err != nil {  
                log.Fatal(err)  
             }  
  
             var mapTaskNum []int  
             for _, f := range files {  
                if strings.HasPrefix(filepath.Base(f), "mr-out-") {  
                   log.Printf("Skipping output file: %s", f)  
                   continue  
                }  
  
                if strings.Contains(f, "worker") || strings.Contains(f, "jobcount") || strings.Contains(f, "tmp") {  
                   log.Printf("Skipping non-intermediate file: %s", f)  
                   continue  
                }  
  
                taskNum, err := ExtractMiddleNum(f)  
                if err != nil {  
                   log.Fatal(err)  
                }  
                mapTaskNum = append(mapTaskNum, taskNum)  
             } //取出所有task编号  
  
             TasksReply, err := QueryTasks()  
             if err != nil {  
                log.Fatal(err)  
             }  
  
             var resultSet []int  
             for _, Num := range mapTaskNum {  
                status, ok := TasksReply.AllTask[Num]  
                if !ok {  
                   log.Fatalf("cannot find task %d", Num)  
                }  
                if status.Status == 1 {  
                   resultSet = append(resultSet, Num)  
                } //得到结果集  
  
             }  
  
             for _, f := range files {  
                if strings.HasPrefix(filepath.Base(f), "mr-out-") {  
                   log.Printf("Skipping output file: %s", f)  
                   continue  
                }  
  
                if strings.Contains(f, "worker") || strings.Contains(f, "jobcount") || strings.Contains(f, "tmp") {  
                   log.Printf("Skipping non-intermediate file: %s", f)  
                   continue  
                }  
                mapNum, err := ExtractMiddleNum(f)  
                if err != nil {  
                   log.Fatal(err)  
                }  
                file, err := os.Open(f)  
                if err != nil {  
                   log.Fatalf("cannot open %v", f)  
                }  
                var dec *json.Decoder  
                for _, num := range resultSet {  
                   if num == mapNum {  
  
                      dec = json.NewDecoder(file)  
  
                   } //得到所有已完成的中间文件的jsondec  
                }  
  
                for {  
                   var kv KeyValue  
                   if err := dec.Decode(&kv); err != nil {  
                      break  
                   }  
                   intermediate = append(intermediate, kv)  
  
                } //得到所有的key value值  
                file.Close()  
             }  
             sort.Sort(ByKey(intermediate))  
             //字典序排序kva  
  
             tempFile, _ := os.CreateTemp(".", "mr-out-tmp-*")  
  
             //  
             // call Reduce on each distinct key in intermediate[],             // and print the result to mr-out-0.             //             i := 0  
             for i < len(intermediate) {  
                j := i + 1  
                for j < len(intermediate) && intermediate[j].Key == intermediate[i].Key {  
                   j++  
                }  
                values := []string{}  
                for k := i; k < j; k++ {  
                   values = append(values, intermediate[k].Value)  
                }  
                output := reducef(intermediate[i].Key, values)  
  
                // this is the correct format for each line of Reduce output.  
                fmt.Fprintf(tempFile, "%v %v\n", intermediate[i].Key, output)  
  
                i = j  
             }  
  
             err = tempFile.Close()  
             if err != nil {  
                return   
}  
             finalName := "mr-out-" + strconv.Itoa(reply.ReduceNum)  
             os.Rename(tempFile.Name(), finalName)  
          }  
       case 3:  
          {  
             log.Println("task over")  
             return  
          }  
       }  
  
       args := CompleteArgs{  
          TaskId:    reply.TaskId,  
          TaskType:  reply.TaskType,  
          FileName:  reply.FileName,  
          ReduceNum: reply.ReduceNum,  
       }  
       _, err = CompleteRequest(args)  
       if err != nil {  
          return  
       }  
  
    }  
  
    // Your worker implementation here.  
  
    // uncomment to send the Example RPC to the coordinator.    // CallExample()  
}
```
## 4. 关键设计点总结

1. **原子操作保障一致性：** 对于 Map 和 Reduce 产出的文件，**一定**要先 `os.CreateTemp` 再 `os.Rename`。这避免了“僵尸 Worker”（跑得慢但没死）写坏已经成功生成的文件。
    
2. **结果过滤机制：** 我在 Reduce 阶段引入了 `QueryTasks` RPC。Worker 会先去问 Master：“磁盘上这些 Map 产出的中间文件，哪些对应的任务 ID 才是官方认证成功的？”。这样就从逻辑上屏蔽了超时任务留下的脏数据。
    
3. **双重状态控制：** Master 负责全局调度（切换 Stage），Worker 负责局部执行。两者通过 RPC 信号保持步调一致。


**总结：**
比较繁琐，因为worker集合了Map和Reduce的逻辑，需要处理Map和Reduce的具体逻辑，需要一些算法处理。

对了，对于Map和Reduce产出的文件一定是需要先os.CreateTemp然后再os.Rename原子命名的，避免worker只是超时而不是崩溃带来的数据竞争问题或者其他隐蔽的问题

其实为了对应Master的各个状态Worker也是一个状态机的形式，读取外部传来的状态，执行对应的逻辑。