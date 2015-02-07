# C++游戏后台异步架构与Lua协程结合探究
**作者: fergus (zfengzhen@gmail.com)**    

`性能、复杂的取舍。`
`程序中一般分为底层事件处理以及业务逻辑处理两大块。`	
`对于业务逻辑这块，经常是变动较大，工作量较多，我们是否可以尝试在不太影响性能的情况下，降低其复杂度？`
###游戏后台中常见的一种架构###
- 异步单进程架构
- 性能较好，可读性差，线性逻辑代码被拆分为许多不同的部分存在不同的代码里，需要根据不同的回调点去串连起来
- 常见的单进程异步代码处理
	- 请求 -> 保存session -> 异步逻辑（其他server服务）
	- 异步逻辑回包 -> 恢复session -> 本地逻辑 -> 保存session -> 异步逻辑（其他server服务）
- session保存的两种方式：
	- 1、session较小，直接保存在网络包上，回包时获取
	- 2、session较大，创建session和id的map，将session保存在共享内存上，网络包上带上session id，回包时根据id获取session

###Task异步模型###
![Task异步模型](https://github.com/zfengzhen/Blog/blob/master/img/lua_async_task_task_model.png)

- Task
	- 多个Step通过链表连接
	- 通过注册定时器完成超时处理，释放资源
	- 负责存储该业务的session数据，并提供接口给Step读取和修改
	- 需要提供以下接口
		- `on_task_timeout` 超时处理接口
		- `on_task_failed` 任务失败时处理接口
		- `on_task_successed` 任务成功完成时处理接口
		- `do_next_step` 当前Step完成时，跳转到下一个Step，提供给Step在Step成功时调用
		- `get_running_step` 获取当前所在的Step
- Step
	- 封装异步操作，一般为SS处理以及数据库操作
	- 需要提供两个接口：
		- `perform`异步任务执行，执行完后回到逻辑主循环
		- `notify` 异步任务执行完后回调
		- `do_complete` 用于Step完成时判断，根据Step的成功失败调用Task的不同操作
- TaskMgr
	- 通过map或者红黑树管理task_id和Task之间的映射，根据回包中的task_id查找到Task，获取到当前所在的Step，调用`Step::notify`
- **优点**：
	- 性能较好
- **缺点**：
	- 逻辑分布在各个Step的perform和notify中，难理解
	- 每个不同任务都得继承Task，实现上述接口；不同的步骤都得继承Step，实现上述接口；代码量较多

###同步模型###
![同步模型](https://github.com/zfengzhen/Blog/blob/master/img/lua_async_task_sync_model.png)

- 模型
	- 采用顺序方式执行逻辑
	- 由于执行IO操作（包括磁盘、网络IO），需要大量时间去等待操作完成，需采用多进程，多线程方式去提高吞吐量
采用同步方式顺序执行逻辑。
	- 由于进程线程的开销，以及线程需要锁的开销，吞吐量比异步模式低不少
- **优点**：
	- 编写容易，代码易读
- **缺点**：
	- 吞吐量较低

###C++与lua异步协程模型###
![C++与lua异步协程模型](https://github.com/zfengzhen/Blog/blob/master/img/lua_async_task_lua_co_model.png)

- AsyncTaskMgr
	- `init` 初始化时，读入所有的lua脚本，新建一个master_state
	- `register_func` 注册lua要用到的所有C函数
	- `create_task` 创建新的任务，通过lua_newthread，根据不同的任务类型，将对应的任务函数放入新建的协程中，并将该协程保存到TASK_TABLE table中
	- `resume` resume相应的任务
	- `close_task` 关闭相应的任务，通过`TASK_TABLE[task_id] = nil`将资源释放交个lua GC管理
- 采用lua协程进行异步逻辑处理
    - 1、C++处理事件触发以及定时器触发
    - 2、根据触发事件找到相应的协程，继续执行
    - 3、在Lua协程中遇到异步逻辑处理时，通过coroutine。yield放弃CPU，恢复到主逻辑
- **优点**：
    - 1、增加异步代码开发效率，所有逻辑按同步方式写入Lua脚本中
    - 2、对于策划易变的逻辑，Lua脚本也会较容易修改，并且能够实现热更新
- 代码: 
	- https://github.com/zfengzhen/lua_async_task