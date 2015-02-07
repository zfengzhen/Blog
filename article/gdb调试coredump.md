# gdb调试coredump  
**作者: fergus (zfengzhen@gmail.com)**    
 
## coredump开启: 

```shell  
ulimit -c unlimited
```

vim /etc/sysctl.conf 添加如下行:  

```shell	
kernel.core_uses_pid = 1
kernel.core_pattern = /tmp/core-%e-%s-%u-%g-%p-%t
fs.suid_dumpable = 2
```

可使用参数:  	
%% – 符号%  
%p – 进程号  
%u – 进程用户id   
%g – 进程用户组id   
%s – 生成core文件时收到的信号   
%t – 生成core文件的 时间 (seconds since 0:00h, 1 Jan 1970)   
%h – 主机名   
%e – 程序文件名   

```
sysctl -p
```
不重启使得/etc/sysctl.conf生效  

## gdb指令:  
bt [backtrace] 打印当前函数调用栈信息    
f[rame] n 切换当前栈, n为从0开始的整数  
info frame 显示当前栈信息  
info args 显示当前函数的参数名及其值  
info locals 显示当前函数所有局部变量及其值  
info catch 显示当前函数中的异常处理信息  

i[nfo] r[egister] ebp 打印ebp  

```shell
ebp 0x232f3e 0x834f2e1
```

x /4x [ebp] 打印ebp指向内容  

```shell
0x232f3e: 0x121a2c 0x1de13c 0x000 0x000
```

info symbol 打印符号  

```shell
__strtoll_internal + 23 in section.text
```
	
ebp的调用信息通过链表组织, 第一个节点为指针, 通过将调用信息连接起来, 紧随气候的4个字节时该层的调用返回地址.  

SIGSEGV段错误(信号11), 访问非法内存, 地址越界, 指针为空  
SIGABRT检测异常(信号6), 调用abort()函数导致, 释放内存再次释放, 内存分配失败等  
SIGBUS总线错误(信号7), 总线地址不存在, 硬件故障  