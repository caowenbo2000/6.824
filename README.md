## QA：

并行运行？多进程还是多线程？同步方式是什么？

并行运行，通过一个协调器组织所有的进程，同步方式使用rpc

woker process 通过请求coordinator分配一个task进行执行

coordinator到关注worker如果未执行成功，则需要把task分配给其他的worker

#### 参考代码 

main/mrcoordinator.go

main/mrworker.go

#### 实现任务：

mr/coordinator.go

mr/worker.go

mr/rpc.go

### rules

* map阶段把不同的key分配到nreduce个buckets中，其中nreduce作为参数传递给MakeCoordinator方法创建nreduce个任务。
* worker的实现需要把第x个任务写进文件mr-out-x中
* 关于mr-out-x的文件格式要使用%v %v的key value形式
* 我们需要修改mr/coordinator.go mr/worker.go mr/rpc.go 这三个文件
* map 的输出需要intermediate输出进文件中，以便后续作为reduce的输入
* 需要在mr/coordinator.go中实现一个done()函数，当所有的任务完成时，这是协调器也会退出
* 当jod完全完成时worker也需要退出，对于该内容一个简单的实现是当woker与coordinator失去联系时可以假定coordinator已经推出，这时任务完成worker也可以退出。

#### Hints

* 一种上手的方式是修改mr/worker.go中的worker()去请求coordinatori一个任务。然后修改协调器去回复一个还未开始的任务，然后修改worker实现读文件调用map函数
* map reduce函数都是在runtime时期使用go plugin load进的
* 更改mr文件夹后要重新编译动态库
* 这个lab依赖于所有的worker全都共享一个文件系统，但是如果要是在多机运行时需要使用GFS
* 一个reasonable的会话名是mr-x-y，x号map任务，y号reduce任务
* maptash需要储存key value存入文件。one way to store is json
* map可以使用ihash函数去pick一个任务给一个指定的key
* 你可以steal一些代码from mrsequential.go 为了读取文件 对piar排序，还有输出文件
* 协调器作为一个rpc server 要处理并发，要对shared data进行加锁，并且使用-race检测
* worker 又是需要等待，例如reduce不能在map没结束时开始，一种可能是worker同时sleep周期性ask task，另一种可能是使用循环检测使用sync.cond
* coordinator不能依靠区分crash的worker，没有crash but stalled for some resaon的 worker和运行太慢的worker，最好的方式就是在等待一定时间后将任务重新分配，在本lab中最好等待10second
* mrapps会随机在map和reduce函数中退出，这样可以验证crash recover的能力
* 为了确保crash不会被感知，mapreduce论文提出一个trick，就是在输出是使用temp文件，在完成时自动rename
* Go Rpc只会发送大写名字的field
* 当想rpc系中传递结构时应该是使用zero-allocated,不要在call的时候给赢得字段设置任何值，否则在初始化reply字段是会使用非默认值你会观察到写看看起来已经生效，但是在调用方非默认值被保留。

