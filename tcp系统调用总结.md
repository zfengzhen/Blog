# tcp系统调用总结  

# send  

	ssize_t send(int sockfd, const void *buf, size_t len, int flags);  
	
## 阻塞模式下:  
send通过阻塞保证发送成功, 除非发送字节数大于该socket的写缓冲区, 如果成功, 则返回的字节数一定等于发送字节数.  
对于错误errno, 需要特别处理EINTR错误, 需要重新发送数据.  

## 非阻塞模式下:  
send即时发送数据, 如果缓冲区如果满了, 则会返回EAGAIN或者EWOULDBLOCK错误, 如果缓冲区有空, 则尽量拷贝, 将要发送数据拷贝到缓冲区中, 返回已经拷贝的数据长度.  

### 需要注意两个问题  
1 碰到EAGAIN或EWOULDBLOCK时重试, 如何重试, 如果重试时间太快, 会继续出现该错误.  
2 如果只发送了部分数据, 如何发送剩下的数据, 如果立马继续发送, 则会出现EAGAIN或EWOULDBLOCK错误  

### 轮询模式下(或者定时器触发)
轮询模式是指主循环每隔一定时间, 循环运行一定的逻辑.  
增加应用层的发送数据缓冲区, 每次发送数据, 将数据追加到数据缓冲区, 轮询或者定时器触发时, 通过send发送数据, 大小为缓冲区上的所有数据
1 如果出现EAGAIN或EWOULDBLOCK则返回, 因为下次轮询或者定时器触发时依然会发送  
2 如果发送了部分数据, 则将已发送的数据移出缓冲区, 下次轮询或者定时器触发会继续发送   
3 如果全部发送成功, 则将所有的数据移出缓冲区, 下次轮询或者定时器触发时, 发送缓冲区数据为空时, 直接返回主循环  

### 事件触发模式下(epoll, libevent)  
事件触发模式监听fd, 一旦可写的时候, 触发相应的事件进行处理  
什么是可写状态?   
LT模式: 可写状态是指socket写缓冲区空闲值达到一定阀值时, 一般这个阀值为1, 也就是写缓冲区空闲超过1字节时就会触发写状态.   
ET模式: unwriteable变为writeable, 也就是得一直写, 出现了EAGAIN以及EWOULDBLOCK时, 就会变成unwriteable, 这样一旦缓冲区变为writeable, 就会触发写状态.  

假设服务器处于LT模式, 需要发送数据时:  
1 将socket加入事件监听, 等待可写事件  
2 触发可写事件时, 发送数据, 如果数据只发送了部分, 从应用层的缓冲区删除已发送的数据, 并直接返回, 下次触发的时候, 依然有数据可写  
3 如果写完, 将socket移出事件监听  
这样的缺点是每次都得将socket加入事件监听, 以及移出事件监听, 有一定的代价.  
优化:  
如果应用层缓冲区没有数据, 则直接发送, 如果遇到EAGAIN或EWOULDBLOCK, 则加入事件监听; 如果部分发送, 则将剩余未发送的数据加入应用层缓冲区, 然后加入事件监听. 毕竟一般都是成功全部发送, 可以减少加入事件监听以及移出事件监听的消耗.  

![image](https://dl.dropboxusercontent.com/s/occtx2y96cjzbka/tcp_system_call_send.jpg)  

# recv
 
	ssize_t recv(int sockfd, void *buf, size_t len, int flags);
	
recv传入参数是希望收到的最大数据量, 返回是实际收到数据量大小. recv工作时, 先检查socket的接收缓冲区是否正在接收数据或者是否有数据, 如果正在接接收数据或者数据未空, 则一直阻塞, 当数据接收完毕的时候, recv把socket缓冲区的数据拷贝到应用层的缓冲区中. 如果socket缓冲区数据大小比应用层的大, 则需要调用多次recv才能接收完整.    
返回0, 则表示对端主动断掉连接.  
(errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN) 认为连接是正常的.  
## 阻塞模式  
recv会阻塞直到有数据到来, 一般单进程异步的情况下不会这么做.  
## 非阻塞模式
可以加入事件监听, 注意LT模式和ET模式的区别. ET模式要读到数据出现EAGAIN的情况才行. 注意自己组包, 拼包处理.  

# accept

	int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

## 阻塞模式
如果没有请求过来, 会阻塞进程.  
## 非阻塞模式
如果没有请求过来, 将会返回EAGAIN或EWOULDBLOCK  
加入事件监听, 触发可读事件, 则表示有新的连接过来.  
当客户在服务器调用accept之前中止某个连接时, accept调用可以立即返回-1, errno设置为ECONNABORTED或者EPROTO错误忽略这两个错误.  

# connect

	int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

tcp三次握手:
	
	client ----SYN j---> server        
	client <---SYN k, ACK j+1--- server          
	client ----ACK k+1---> server        

## 阻塞模式
1 SYN响应过满, 阻塞, 直到阻塞模式超时.  
2 如果对客户端的SYN响应RST, 立即返回ECONNREFUSED错误.  
3 如果发出的SYN在中间的路由器上引发了一个目的地不可达ICMP错误, 多次尝试发送失败后返回错误号为ENETUNREACH.  

## 非阻塞模式
直接调用connect会返回一个EINPROGRESS错误, 但不意味着连接出错.  
加入监听事件(epoll select libevent), 当连接成功时, 返回socket可写; 单连接失败时, 返回socket可读可写.  
正确的作法, 当返回socket可写时, 需要调用getsockopt来判断连接是否成功.  

	int error = 0;
	unsigned int len = sizeof(error);
	if (getsockopt(socket, SOL_SOCKET, SO_ERROR, &error, &len) == 0) {
		// 建立成功
		return 0;
	}