# AI实现思考
**作者: fergus (zfengzhen@gmail.com)**    

项目中采用行为树[behaviac](https://github.com/TencentOpen/behaviac)+状态机的形态来做AI.  
![image](https://github.com/zfengzhen/Blog/blob/master/img/ai_behaviac.png)  
行为树主要做AI层面上的决策, 每次决策都会运行到行为树底层的Action节点.  
Action节点上的三种逻辑:  
- 执行逻辑函数, 比如Move()  
- 赋值操作, 比如target = xxxxx  
- 切换行为树(决策)  

执行逻辑函数包括了状态机的切换, 状态机中不包含状态切换的代码, 状态切换通过行为树(决策)来进行, 状态机只包含状态的相关参数, 比如移动状态只包括移动相关的信息, 比如移动速度, 移动方向, 设置到外部的移动模块, 传递相应的参数, 移动模版通过tick或者定时器进行驱动.  

behaviac提供了AI编辑器, 也就是说程序只要把AI底层的原子逻辑封装好, 提供给behaviac, 通过behaviac编辑器进行拖动各种AI就可以导出相应的AI, 程序运行时加载相应的AI, 并挂载到相应的实体上, 就可以运行.  

我们把上面的AI框架进行抽象:  
AI = **决策** + **动作**  
![image](https://github.com/zfengzhen/Blog/blob/master/img/ai_impl.png)  
决策就是逻辑判定, 动作就是AI逻辑的具体执行  
上面我们决策这块采用的是behaviac行为树来实现, 我们也可以用其他方式来实现  
根据决策是否需要返回running状态(也就是说决策逻辑是否需要挂起, 下一次tick进来的时候, 能直接在上次挂起的逻辑上跑)来看看有其他哪些实现方式  

##### 需要返回running状态的情况需要**协程**的方式来实现  
- 1 lua以及lua协程  
- 2 c++通过swapcontext进行封装  

1 lua以及lua协程, 需要向lua中注册相应的C++函数, 该方式比较好的就是能够支持热更新, 每个实体绑定一个lua协程  
2 c++封装的协程比较好的就是不需要注册函数的过程, 不过协程栈大小的判定需要经验值, 我们尽量不在协程栈中分配变量, 通过实体指针的方式处理, 协程中只做逻辑  

#### 不需要running状态的情况  
直接用c++的if else switch 进行决策逻辑处理, 简单直接, 不用running状态的情况就会每次决策都得从头到尾的跑一遍, 这样需要在决策逻辑中建立多级的if else的判定, 尽量少跑判定逻辑  

---  