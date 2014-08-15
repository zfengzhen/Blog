# hold住你的后台程序

## 简介  
*  hold住你的后台程序，心中自然无码
	*  1、了解常用系统调用和库函数量级
	*  2、清楚你编写的每个函数的大致性能
	*  3、掌握一门工具分析程序的瓶颈 
  
---

## 系统调用和库函数的量级  

系统调用和库函数的量级依赖于操作系统以及机器硬件配置   
本文数据测试基于测试数据基于：suse64、16核CPU、64G内存、300G硬盘、千兆网卡  

### 内存：write
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_memory_write.jpg)  
  
### 硬盘：write、read
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_disk_write_read.jpg)  
  
### 硬盘读写的背后
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_%E7%A3%81%E7%9B%98%E8%AF%BB%E5%86%99%E8%83%8C%E5%90%8E.jpg)  
  
### 文件打开关闭：open close
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_open_close.jpg)  
  
### 时间戳函数：time gettimeofday
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_time.jpg)  
  
### VDSO
* Virtual Dynamic Shared Object
* 虚拟动态共享对象
* 将内核函数映射到用户空间，虚拟了一个so共享模块`linux-vdso.so.1`，这个模块实际上不存在，调用起来和直接调用用户空间的C函数一样，减少了系统调用开销。VDSO经常用来用于给gettimeofday提供快速访问。   
  
### time gettimeofday vdso加速
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_time_vdso.jpg)  
  
### VDSO启动参数
* /proc/sys/kernel/vsyscall64
	* 0 提供最精确的微秒时间间隔解决方案，但是也是开销最大，因为它使用了一个常规的系统调用
	* 1 稍微没那么精确，但也是微秒级别的，较低的开销
	* 2 最不精确，毫秒级别的时间间隔，但是最低的开销
  
### 网络相关系统调用  
![](https://github.com/zfengzhen/Blog/blob/master/img/hold%E4%BD%8F%E4%BD%A0%E7%9A%84%E5%90%8E%E5%8F%B0%E7%A8%8B%E5%BA%8F_%E7%BD%91%E7%BB%9C%E7%9B%B8%E5%85%B3%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8.jpg)  
  
总结：碰到自己常用的系统调用和库函数可以自己测一测，掌握这些常用系统调用和库函数的大概量级

---

##函数性能评估
清楚你编写的每个函数的大致性能   
  
### gprof
* gprof使用
	* 用编译器对程序进行编译，加上-pg参数
	* 运行编译后的程序，生成gmon.out文件
	* 用gprof工具分析gmon.out
* 优点：
	* 简单易用
* 缺点：
	* 不太适合后台程序
	* 只能分析程序运行过程中消耗的用户时间，无法得到程序内核空间的运行时间  
  
### 利用共享内存以及C++构造析构函数进行函数性能分析
* 获取函数单次运行时长
	* 在函数的开头实例化一个特殊的类
	* 在该类构造函数和析构函数中调用gettimeofday
	* 相减得到耗时
* 存储该耗时到共享内存，并求多次平均值
	* avg_cost += (cost – avg_cost)/call_times;
* 编写一个工具读取共享内存数据，并进行分析 
	* 优点：能够得到用户级别函数的性能，压测时能够得到后台程序的执行性能
	* 缺点：需要编写实现；单进程数据比较准确；多进程时只能通过总体开始时间以及结束时间进行计算
  
---
  
##系统性能瓶颈评估
  
### perf  
linux系统性能分析工具（内核2.6.3x及以上），应用程序可以利用`PMU，tracepoint和内核中的特殊计数器`来进行性能统计，不但可以分析应用程序性能问题，还可以分析内核性能问题，从而全面理解应用程序中性能瓶颈。  
参考:        
[Perf -- Linux下的系统性能调优工具，第 1 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-perf1/)     
[Perf -- Linux下的系统性能调优工具，第 2 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-perf2/)      
  
* perf list
	* 功能：查看当前环境支持的性能事件
* perf stat
	* 功能：收集性能数据统计信息
	* 常用参数：
		* -e <event> : 指定性能事件
		* -p <n> : 指定进程PID
		* -d : 更详细的性能事件
* perf record
	* 功能：收集一段事件内的性能数据
	* 常用参数：
		* -e <event> : 指定性能事件
		* -p <n> : 指定进程PID
		* -a : 分析整个系统的性能
		* -f : 强制覆盖之前的性能文件
		* -g : 收集函数间的调用关系
		* -o <file>: 指定性能数据文件名字
* perf report
* 功能：分析性能数据
	* 常用参数：
		* -i <file> : 读入的性能分析数据
		* -g : 输出调用关系
		* -v : 显示每个符号的地址
		* -n : 显示每个符号对应的事件数
* perf top
	* 功能：实时显示系统性能数据
	* 常用参数：
		* -e <event> : 指定性能事件
		* -p <n> : 指定进程PID
		* -d : 界面刷新周期
		* -K : 不显示内核符号
		* -U : 不显示用户符号

### gprof2dot
* [gprof2dot](http://code.google.com/p/jrfonseca/wiki/Gprof2Dot)是一个python脚本把各种性能分析工具的的输出转换成**dot**格式文件，支持prof、gprof、Linux perf、oprofile、Valgrind's callgrind tool等。
* 用法：`gprof2dot.py [options] [file] ...`
	* 常用参数：
	* -o FILE : 输出文件（默认stdout）
	* -e PERCENTAGE : 删除权值在PERCENTAGE以下的边（默认0.1）
	* -f FORMAT : 指定性能分析文件格式profile format: prof, callgrind, oprofile, hprof,sysprof, sleepy, aqtime, pstats, axe, **perf**, or xperf（默认prof）
  
### graphviz
* [Graphviz](http://www.graphviz.org/)由一种被称为DOT语言的图形描述语言与一组可以生成和/或处理DOT文件的工具组成。
* dot ：一个用来将生成的图形转换成多种输出格式的命令行工具。其输出格式包括PostScript，PDF，SVG，PNG，含注解的文本等等。
* 用法：`dot –Tpng xxx.dot –o pic.png`
   
### 性能分析步骤
* 1、产生perf性能文件
	* `perf record –afg –p pid –o perf.data.pid sleep 10`
	* http://linux.die.net/man/1/perf-record
* 2、查看perf.data文件
	* `perf report –i perf.data.pid`
	* http://linux.die.net/man/1/perf-report
* 3、将perf.data文件转换成dot文件（python 2.6以上）
	* `perf script –i perf.data.pid | python gprof2dot.py –f perf –e 1 –o xxx.dot`
	* http://linux.die.net/man/1/perf-script
	* http://code.google.com/p/jrfonseca/wiki/Gprof2Dot
* 4、将xxx.dot文件转换成png图片
	* `dot –Tpng xxx.dot –o pic.png`
	* http://www.graphviz.org/pdf/dotguide.pdf
  
### 性能结果
![](https://github.com/zfengzhen/Blog/blob/master/img/03o6s3fzlofbj7w/perf_graph.png)

> 从生成的结果图去寻址程序或者系统性能瓶颈。