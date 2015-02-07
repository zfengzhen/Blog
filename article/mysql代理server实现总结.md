# mysql代理server实现总结  
**作者: fergus (zfengzhen@gmail.com)**    

## 需求:
假设某个逻辑server为单进程异步模式, 在这种情况下不能直接去访问mysql数据库, 否则会阻塞逻辑server. 这个时候需要单独拉出一个msyql的代理server, 使得逻辑server通过协议方式去访问mysql代理server, 实现数据库的异步交互.(后续mysql代理server称为**datasvr**, 逻辑server称为**logicsvr**)

## 1 单进程单连接  
在业务量不大的情况下, datasvr采用单进程单连接的方式去访问mysql, 实现简单以及可以快速完成.  

## 2 多进程且每个进程有一个独立连接  
业务量上来的情况下, datasvr采用多进程部署, 每个进程采用单进程单连接的方式, 这个时候需要有一个代理进程去负责分包路由, 代理进程和工作进程的通信可以采用UDP或者是共享内存队列(看业务情况), 分包路由可以采用简单的按工作进程数取余数分发给工作进程, 或者是通过工作进程进行抢占式工作.  

## 3 多线程, 连接池, 每来一个请求创建一个线程, 并从连接池中获取一个空闲连接
初始化一个连接池, 连接池的实现采用单例模式, 当收到logicsvr的请求时, 创建一个线程执行相应的函数, 不同的数据库操作对应不同的函数, 并从连接池中获取一个空闲连接, 进行数据库操作, 当查询结束, 给logicsvr回响应包后, 线程销毁. 线程的频繁创建和销毁会损耗一部分性能.  

## 3 多线程, 主线程负责网络收发包, 工作线程负责数据库操作  
分配两个队列: **网络请求队列**, **数据库结果队列**  
主线程收到网络请求后, 将相应的请求压入网络请求队列, 工作线程轮询(或者更好的方式select, epoll)网络请求队列, 一旦有数据就开始读取数据进行数据库查询, 并把查询结果放入数据库结果队列. 主线程轮询数据库结果队列, 如果有数据则处理数据并进行相应的回包.     
这种模式实际上类似于单进程, 只是缓存了请求包, 如果logicsvr跟datasvr采用共享内存队列的工作方式, 实际上缓存在共享内存上, 可以采用单进程单连接的模式.   
![](https://github.com/zfengzhen/Blog/blob/master/img/mysql_proxy_server_2thread.jpg)  

## 4 多线程, 每个线程绑定一个数据库连接, 有一个同一的管理类来管理这些数据库连接, 进行分配, 回收等工作  
数据库连接包含在MysqlClient类中, MysqlClient还绑定了一个存储sql语句的缓存, 一个执行工作的线程, 一对pipe的fd, 一个数据库结果集, 数据库相关的属性设置, 一个回调函数.  
MysqlClient绑定的工作线程的主循环: select pipefd 看是否有数据写入, 如果有则执行相应的sql语句, 执行完后调用回调函数.  
MysqlClient的主线程: 如果执行sql操作时, 写入sql到相应的字段, 写入回调函数到相应的字段, 并写入pipe一个字节, 用来通知工作线程.  
MysqlMgr用来管理多个MysqlClient, 维持一个空闲MysqlClient队列, 每次从中取出一个空闲的MysqlClient, 并执行相应的sql查询, 注册回调函数.  
框架:  
![](https://github.com/zfengzhen/Blog/blob/master/img/mysql_proxy_server_mysqlmgr.jpg)  
执行流程:  
![](https://github.com/zfengzhen/Blog/blob/master/img/mysql_proxy_server_mutithread.jpg)  

