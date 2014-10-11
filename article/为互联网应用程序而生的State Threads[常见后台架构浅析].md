# 为互联网应用程序而生的State Threads[常见后台架构浅析]  

本文翻译来自[State Threads介绍](http://state-threads.sourceforge.net/docs/st.html)  

### Introduction  
### 介绍  

    State Threads is an application library which provides a foundation for writing fast and highly scalable Internet Applications on UNIX-like platforms. It combines the simplicity of the multithreaded programming paradigm, in which one thread supports each simultaneous connection, with the performance and scalability of an event-driven state machine architecture.  

State Threads是一个应用库：为性能得更快以及高可伸缩性的类UNIX平台的互联网应用程序提供基础库。它拥有多线程编程范式的简单（一个线程支持一个并发连接）以及事件驱动状态机架构的性能和可伸缩性。

# 1. Definitions  
# 1. 定义  

## 1.1 Internet Applications
## 1.1 互联网应用程序  

    An Internet Application (IA) is either a server or client network application that accepts connections from clients and may or may not connect to servers. In an IA the arrival or departure of network data often controls processing (that is, IA is a data-driven application). For each connection, an IA does some finite amount of work involving data exchange with its peer, where its peer may be either a client or a server. The typical transaction steps of an IA are to accept a connection, read a request, do some finite and predictable amount of work to process the request, then write a response to the peer that sent the request. One example of an IA is a Web server; the most general example of an IA is a proxy server, because it both accepts connections from clients and connects to other servers.  

一个*`互联网应用程序（IA）`*是一个客户端或服务器端网络应用，它接收来自客户端的连接，也许也需要连接到服务器。在*IA*中，数据的到达和发送通常控制着处理逻辑（也就是说*IA*是一个数据驱动型应用）。对于每一个连接，一个*IA*做着有限数量的工作，包括跟对等端（可能是一个客户端或者服务器端）交换数据。一个*IA*的典型的处理步骤：接受一个连接，读取一个请求，做一些有限且可预估数量的工作去处理这个请求，然后回包到发送请求的对等端。WEB服务就是一个*IA*的例子，更典型的*IA*例子就是代理服务器，因为它需要接受来自客户端的连接，也需要连接到其他的服务器。  

    We assume that the performance of an IA is constrained by available CPU cycles rather than network bandwidth or disk I/O (that is, CPU is a bottleneck resource).  

我们假定*IA*的性能受限于可用的CPU周期，而不是网络带宽或者是磁盘I/O（也就是说，CPU是瓶颈资源）。

## 1.2 Performance and Scalability
## 1.2 性能和柔性  

    The performance of an IA is usually evaluated as its throughput measured in transactions per second or bytes per second (one can be converted to the other, given the average transaction size). There are several benchmarks that can be used to measure throughput of Web serving applications for specific workloads (such as SPECweb96, WebStone, WebBench). Although there is no common definition for scalability, in general it expresses the ability of an application to sustain its performance when some external condition changes. For IAs this external condition is either the number of clients (also known as "users," "simultaneous connections," or "load generators") or the underlying hardware system size (number of CPUs, memory size, and so on). Thus there are two types of scalability: load scalability and system scalability, respectively.  

评估*IA*是以每秒处理任务数或每秒处理的字节数（如果给出平均的任务大小，其中一个可以转换成另一个）的吞吐量来衡量的。有许多基准测试程序能够用来衡量WEB服务应用的特定负载的吞吐量（比如SPECweb96, WebStone, WebBench）。虽然*`可伸缩性(scalability)`*没有通用的定义，*一般来说它表示应用程序在外部条件变化时维持其性能的能力*。*IA*的外部条件可能是客户端的个数（或者是说用户个数，并发连接数或者负载生成源）或者底层硬件系统的配置（CPU的个数、内存的大小等等）。因此这有两种类型的可伸缩性：*`负载可伸缩性`*和*`系统可伸缩性`*。
（**译注**：scalability翻译成柔性，可伸缩性，表示规模可调整；extensibility翻译成可扩展性，表示应用是否设计得比较好、模块化是否抽象得好，当添加新功能时能够比较容易）  

    The figure below shows how the throughput of an idealized IA changes with the increasing number of clients (solid blue line). Initially the throughput grows linearly (the slope represents the maximal throughput that one client can provide). Within this initial range, the IA is underutilized and CPUs are partially idle. Further increase in the number of clients leads to a system saturation, and the throughput gradually stops growing as all CPUs become fully utilized. After that point, the throughput stays flat because there are no more CPU cycles available. In the real world, however, each simultaneous connection consumes some computational and memory resources, even when idle, and this overhead grows with the number of clients. Therefore, the throughput of the real world IA starts dropping after some point (dashed blue line in the figure below). The rate at which the throughput drops depends, among other things, on application design.  

下图展示了一个理想化的*IA*当客户端数量的增长时（蓝色实线）吞吐量的变化。初始阶段吞吐量的增长表现为线性（斜率表示一个客户端时能个提供的最大吞吐量）。在初始阶段，*IA*未充分利用且CPU有部分闲置。进一步的提高客户端的数量会导致系统饱和，吞吐量逐渐停止增长直到CPU满载。在这个点后，吞吐量保持平坦，因为没有更多的CPU周期可用。在现实的环境中，每一个并发连接（甚至当它空闲时）都要消耗一些计算和内存资源，这些开销随着客户端的数量增长而增长。因此，*IA*的现实环境的吞吐量在客户端数量增长到某个点后开始下降（图中的蓝色虚线）。吞吐量的下降速率取决于应用程序的设计。  

    We say that an application has a good load scalability if it can sustain its throughput over a wide range of loads. Interestingly, the SPECweb99 benchmark somewhat reflects the Web server's load scalability because it measures the number of clients (load generators) given a mandatory minimal throughput per client (that is, it measures the server's capacity). This is unlike SPECweb96 and other benchmarks that use the throughput as their main metric (see the figure below).  

我们设定一个应用程序有优秀的*负载可伸缩性*是指它能够在较大负载范围内维持它的*`吞吐量`*。有趣的是， SPECweb99基准测试程序某种程度反映了WEB服务的*负载可伸缩性*，因为它测量了平均每个客户端强制性最小吞吐量时(**译注**:客户端不卡时的所需要的吞吐量)的客户端的数量（也就是它测量了服务器的能力）。 SPECweb96 和其他基准测试程序主要时用吞吐量来作为它们的主要指标（见下图）。
 
![](https://github.com/zfengzhen/Blog/blob/master/img/%E4%B8%BA%E4%BA%92%E8%81%94%E7%BD%91%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E8%80%8C%E7%94%9F%E7%9A%84State_Threads_fig.gif)  

    System scalability is the ability of an application to sustain its performance per hardware unit (such as a CPU) with the increasing number of these units. In other words, good system scalability means that doubling the number of processors will roughly double the application's throughput (dashed green line). We assume here that the underlying operating system also scales well. Good system scalability allows you to initially run an application on the smallest system possible, while retaining the ability to move that application to a larger system if necessary, without excessive effort or expense. That is, an application need not be rewritten or even undergo a major porting effort when changing system size.  

*系统可伸缩性*是指应用程序每单位硬件（比如CPU）能够增加的性能。换句话说，优秀的系统可伸缩性意味着增加一倍的处理器能够大致使得应用程序的吞吐量增加一倍（绿色虚线）。我们假定底层的操作系统也有不错的可伸缩性。优秀的系统可伸缩性允许你在一个最小的系统配置上运行应用程序的可能，同时保留在有必要时不花费过多的力气去高系统配置上运行的能力。也就是说，当改变系统配置时应用程序不需要重写以及移植工作。  

    Although scalability and performance are more important in the case of server IAs, they should also be considered for some client applications (such as benchmark load generators).  

虽然可伸缩性和性能在*IA*上更重要，客户端程序（比如基准测试负载生成源）也需要考虑。   

## 1.3 Concurrency
## 1.3 并发  

    Concurrency reflects the parallelism in a system. The two unrelated types are virtual concurrency and real concurrency.  

并发反映了系统的并行性。有两种不太相关的类型：*`虚拟并发`*和*`物理并发`*。    

    • Virtual (or apparent) concurrency is the number of simultaneous connections that a system supports.   
    • Real concurrency is the number of hardware devices, including CPUs, network cards, and disks, that actually allow a system to perform tasks in parallel.  
•	虚拟并发是指系统同时支持的连接数。   
•	物理并发是指硬件设备的数量，包括CPU、网卡、和硬盘，允许系统真正意义上并行的执行任务。    

    An IA must provide virtual concurrency in order to serve many users simultaneously. To achieve maximum performance and scalability in doing so, the number of programming entities than an IA creates to be scheduled by the OS kernel should be kept close to (within an order of magnitude of) the real concurrency found on the system. These programming entities scheduled by the kernel are known as kernel execution vehicles. Examples of kernel execution vehicles include Solaris lightweight processes and IRIX kernel threads. In other words, the number of kernel execution vehicles should be dictated by the system size and not by the number of simultaneous connections.  

一个*IA*必须提供虚拟并发去同时服务许多用户。为了达到最大的性能和可伸缩性，*IA*创建的被操作系统调度的编程实体应该和系统的物理并发保持一致。被内核调度的编程实体叫做*`内核执行载体`*。内核执行载体的例子包括Solaris lightweight进程和IRIX内核线程。换一种说法，**内核执行载体的数量应该由系统的硬件决定而不是并发连接数**。（进程数由CPU决定而不是并发连接数）

# 2. Existing Architectures  
# 2. 现有的架构  

    There are a few different architectures that are commonly used by IAs. These include the Multi-Process, Multi-Threaded, and Event-Driven State Machine architectures.   

*IA*有一些不同的常用架构。包括多进程架构、多线程架构和事件驱动状态机架构。

## 2.1 Multi-Process Architecture
## 2.1 多进程架构

    In the Multi-Process (MP) architecture, an individual process is dedicated to each simultaneous connection. A process performs all of a transaction's initialization steps and services a connection completely before moving on to service a new connection.  

在多进程架构中，一个独立的进程用来服务一个并发连接。一个进程执行所有任务的初始化步骤，并在服务一个新的连接前完全服务于上一个连接。  

    User sessions in IAs are relatively independent; therefore, no synchronization between processes handling different connections is necessary. Because each process has its own private address space, this architecture is very robust. If a process serving one of the connections crashes, the other sessions will not be affected. However, to serve many concurrent connections, an equal number of processes must be employed. Because processes are kernel entities (and are in fact the heaviest ones), the number of kernel entities will be at least as large as the number of concurrent sessions. On most systems, good performance will not be achieved when more than a few hundred processes are created because of the high context-switching overhead. In other words, MP applications have poor load scalability.  

*IA*的用户*`会话(session)`*相对独立；进程处理不同连接时，没有*`数据同步`*的必要。因为每个进程都有它自己的地址空间，这个架构非常健壮。如果服务一个连接的进程崩溃了，其他的会话不会被影响。然而，为了服务许多并发的连接，将创建同等数量的进程。由于进程是内核实体（并且实际上是最重的一种），为了使得服务正常，内核实体的数量至少要比并发会话的数量大。在大多数系统中，由于上下文切换的开销，创建成百上千的进程时将不会有优秀的性能。换句话说，多进程应用程序负载可伸缩性较低。  

    On the other hand, MP applications have very good system scalability, because no resources are shared among different processes and there is no synchronization overhead.  

另外一方面，多进程应用程序拥有优秀的系统可伸缩性，由于不同的进程没有资源共享，没有*数据同步*的开销。  

    The Apache Web Server 1.x ([Reference 1]) uses the MP architecture on UNIX systems.

Apache服务器1.x在UNIX系统中采用多进程架构。

## 2.2 Multi-Threaded Architecture
## 2.2 多线程架构  

    In the Multi-Threaded (MT) architecture, multiple independent threads of control are employed within a single shared address space. Like a process in the MP architecture, each thread performs all of a transaction's initialization steps and services a connection completely before moving on to service a new connection.  

在多线程架构中，多个独立线程共享一个地址空间。就像在多进程架构中的进程，每个线程执行所有的任务初始化步骤并且在服务下一个连接前完全服务于上一个连接。  

    Many modern UNIX operating systems implement a many-to-few model when mapping user-level threads to kernel entities. In this model, an arbitrarily large number of user-level threads is multiplexed onto a lesser number of kernel execution vehicles. Kernel execution vehicles are also known as virtual processors. Whenever a user-level thread makes a blocking system call, the kernel execution vehicle it is using will become blocked in the kernel. If there are no other non-blocked kernel execution vehicles and there are other runnable user-level threads, a new kernel execution vehicle will be created automatically. This prevents the application from blocking when it can continue to make useful forward progress.  

大多数的现在UNIX操作系统实现了一个多对一的模型，用来映射用户态线程空间到*内核实体*。在这种模型中，任意数量的用户态线程复用到较少数量的*内核执行载体*。*内核执行载体*也被称为虚拟处理器。当一个用户态线程执行了一个阻塞的系统调用，内核执行载体将在内核中阻塞。如果没有其他非阻塞的内核执行载体，并且此时有其他已经就绪的用户态线程，一个新的*内核执行载体*将会自动创建。这样防止应用程序阻塞。  

    Because IAs are by nature network I/O driven, all concurrent sessions block on network I/O at various points. As a result, the number of virtual processors created in the kernel grows close to the number of user-level threads (or simultaneous connections). When this occurs, the many-to-few model effectively degenerates to a one-to-one model. Again, like in the MP architecture, the number of kernel execution vehicles is dictated by the number of simultaneous connections rather than by number of CPUs. This reduces an application's load scalability. However, because kernel threads (lightweight processes) use fewer resources and are more light-weight than traditional UNIX processes, an MT application should scale better with load than an MP application.  

由于*IA*性质上是网络I/O驱动，所有并发会话会阻塞在网络I/O的各个地方。因此，在内核中创建的虚拟处理器的个数接近于用户态的线程数（或者并发连接数）。这样的话，多对一模型退化为一对一模型。就像在多进程架构中，*内核执行载体*的数量取决于并发连接数，而不是CPU数量。这样减少了应用程序的负载可伸缩性。然后，由于内核线程使用较少的资源并且比传统的UNIX进程更轻，一个多线程应用程序比多进程应用程序在负载可伸缩性上更好。

    Unexpectedly, the small number of virtual processors sharing the same address space in the MT architecture destroys an application's system scalability because of contention among the threads on various locks. Even if an application itself is carefully optimized to avoid lock contention around its own global data (a non-trivial task), there are still standard library functions and system calls that use common resources hidden from the application. For example, on many platforms thread safety of memory allocation routines (malloc(3), free(3), and so on) is achieved by using a single global lock. Another example is a per-process file descriptor table. This common resource table is shared by all kernel execution vehicles within the same process and must be protected when one modifies it via certain system calls (such as open(2), close(2), and so on). In addition to that, maintaining the caches coherent among CPUs on multiprocessor systems hurts performance when different threads running on different CPUs modify data items on the same cache line.  

不幸的是，小部分虚拟处理器在多线程架构中共享相同的地址破坏了应用的*系统可伸缩性*，因为线程的竞争导致争抢各种锁。甚至应用程序花心思去优化，避免连接在全局变量有锁竞争，但是仍然有标准库函数和系统调用访问隐藏在应用程序中的相同的资源。比如，在大多数平台，线程的内存分配函数（malloc(3),free(3)等）都是用了一个单独的全局锁。另一个例子是进程的文件描述表。这个通用的资源表在同一个进程中被所有的*内核执行载体*共享，当通过特定的系统调用（比如open(2), close(2)等）修改它时都需要被保护。除此之外，多处理器系统中维持缓存的一致性，当不同的线程跑在不同的CPU上却在相同的cache line中修改数据都会严重降低性能。

    In order to improve load scalability, some applications employ a different type of MT architecture: they create one or more thread(s) per task rather than one thread per connection. For example, one small group of threads may be responsible for accepting client connections, another for request processing, and yet another for serving responses. The main advantage of this architecture is that it eliminates the tight coupling between the number of threads and number of simultaneous connections. However, in this architecture, different task-specific thread groups must share common work queues that must be protected by mutual exclusion locks (a typical producer-consumer problem). This adds synchronization overhead that causes an application to perform badly on multiprocessor systems. In other words, in this architecture, the application's system scalability is sacrificed for the sake of load scalability.  

为了提高*负载可伸缩性*，一些应用程序采用了不同类型的多线程架构：为每个任务创建一个或更多的线程，而不是每个连接去创建一个线程。比如，一个小的线程组用于接收客户端连接，一些线程组负责请求处理，另外一些线程组处理回包。这个架构的最主要优点是解除了线程数量和并发连接数之间的紧密耦合。然而，在这种架构中，不同的特定任务线程组必须共享共同的工作队列，他们必须被互斥锁保护（典型的生产者-消费者问题）。这样添加了*数据同步*的开销导致应用程序在多处理器系统中执行很糟糕。换句话，在这种架构中，应用程序的用系统可伸缩性换取了负载可伸缩性。  

    Of course, the usual nightmares of threaded programming, including data corruption, deadlocks, and race conditions, also make MT architecture (in any form) non-simplistic to use.

当然，多线程编程的通常的噩梦，包括数据破坏，死锁，和竞态条件，并且使得多线程架构的使用不是那么简单。  

## 2.3 Event-Driven State Machine Architecture
## 2.3 事件驱动状态机架构  

    In the Event-Driven State Machine (EDSM) architecture, a single process is employed to concurrently process multiple connections. The basics of this architecture are described in Comer and Stevens [Reference 2]. The EDSM architecture performs one basic data-driven step associated with a particular connection at a time, thus multiplexing many concurrent connections. The process operates as a state machine that receives an event and then reacts to it.  

在*`事件驱动状态机架构(EDSM)`*中，一个进程被用来并发处理多个连接。架构的基本概念在Comer和Stevens [Reference 2]中有描述。*EDSM*在同一时间内特定连接中的一个基本的数据驱动步骤，因此可以复用许多并发的连接。进程作为一个*状态机*，收到一个事件然后对其做出反应。  

    In the idle state the EDSM calls select(2) or poll(2) to wait for network I/O events. When a particular file descriptor is ready for I/O, the EDSM completes the corresponding basic step (usually by invoking a handler function) and starts the next one. This architecture uses non-blocking system calls to perform asynchronous network I/O operations. For more details on non-blocking I/O see Stevens [Reference 3].  

在*EDSM*的空闲状态，调用select(2)或者poll(2)等待网络I/O事件。当一个特定的fd可以读写时，*EDSM*完成相应的基本步骤（通常是调用一个处理函数）并开始下一个。这个架构使用非阻塞的系统调用去执行异步的网络I/O操作。更多关于非阻塞I/O的详细结束参见Stevens [Reference 3]。  

    To take advantage of hardware parallelism (real concurrency), multiple identical processes may be created. This is called Symmetric Multi-Process EDSM and is used, for example, in the Zeus Web Server ([Reference 4]). To more efficiently multiplex disk I/O, special "helper" processes may be created. This is called Asymmetric Multi-Process EDSM and was proposed for Web servers by Druschel and others [Reference 5].  

利用*`硬件的并行性`*（物理并行），可以创建多个相同的进程。这个被称为*`对称多进程EDSM架构`*，例如，Zeus Web服务([Reference 4])。为了更有效的复用磁盘I/O，可以创建“辅助”进程。这个称为*`非对称多进程EDSM架构`*，在Drushchel的WEB服务器和其他的一些服务被提出[Reference 5]。  

    EDSM is probably the most scalable architecture for IAs. Because the number of simultaneous connections (virtual concurrency) is completely decoupled from the number of kernel execution vehicles (processes), this architecture has very good load scalability. It requires only minimal user-level resources to create and maintain additional connection.  

*EDSM*可能是互联网应用中最可伸缩的架构。由于并发连接（虚拟并发）的数量完全脱离内核执行载体（进程）的数量，这个架构有最好的负载可伸缩性，它仅需要最少的用户态资源去创建和维持额外的连接。  

    Like MP applications, Multi-Process EDSM has very good system scalability because no resources are shared among different processes and there is no synchronization overhead.  

像多进程架构应用一样，多进程*EDSM*有很好的系统可伸缩性，由于不同的进程没有共享资源，没有*数据同步*开销。  

    Unfortunately, the EDSM architecture is monolithic rather than based on the concept of threads, so new applications generally need to be implemented from the ground up. In effect, the EDSM architecture simulates threads and their stacks the hard way.

不幸的是，*EDSM*是一套新的整体，而不是基于线程的概念，所以新的应用程序需要从头开始实现。实际上，*EDSM*采用很复杂的方式模拟线程和它们的栈。

# 3. State Threads Library
# 3. State Threads库

    The State Threads library combines the advantages of all of the above architectures. The interface preserves the programming simplicity of thread abstraction, allowing each simultaneous connection to be treated as a separate thread of execution within a single process. The underlying implementation is close to the EDSM architecture as the state of each particular concurrent session is saved in a separate memory segment.

State Threads库采用了上面架构的所有优点。接口保持了和线程编程的简单，允许每一个并发连接就像在进程中的一个单独的线程处理一样。底层的实现就接近于*EDSM*，每个特定的并发会话状态存储于一个隔离的内存段。

## 3.1 State Changes and Scheduling
## 3.1 状态改变和调度  

    The state of each concurrent session includes its stack environment (stack pointer, program counter, CPU registers) and its stack. Conceptually, a thread context switch can be viewed as a process changing its state. There are no kernel entities involved other than processes. Unlike other general-purpose threading libraries, the State Threads library is fully deterministic. The thread context switch (process state change) can only happen in a well-known set of functions (at I/O points or at explicit synchronization points). As a result, process-specific global data does not have to be protected by mutual exclusion locks in most cases. The entire application is free to use all the static variables and non-reentrant library functions it wants, greatly simplifying programming and debugging while increasing performance. This is somewhat similar to a co-routine model (co-operatively multitasked threads), except that no explicit yield is needed -- sooner or later, a thread performs a blocking I/O operation and thus surrenders control. All threads of execution (simultaneous connections) have the same priority, so scheduling is non-preemptive, like in the EDSM architecture. Because IAs are data-driven (processing is limited by the size of network buffers and data arrival rates), scheduling is non-time-slicing.  

每个并发会话的状态包括它的*`栈环境（栈指针，程序计数器，CPU寄存器）和栈`*。从概念上讲，一个线程的上下文切换可以视为进程改变状态。除了进程之外没有调用任何*内核实体*。不像通常意义上的线程库，State Threads库是完全自我控制的。线程的上下文切换（进程的状态）只能在明确的一些函数中触发（比如I/O操作和明确的数据同步）。这样的话，进程的全局变量在大部分情况下不需要被互斥锁保护。整个应用程序可以随意使用所有的静态变量和不可重入的库函数，在增加性能的同时极大的简化编程和调试。这个其实和*`协程模型`*类似，除了不需要显示的调用yield，线程执行一个阻塞的I/O操作并交出控制权。所有的线程的执行（并发连接）有相同的优先级，因此调度是无优先的，就像*EDSM*。由于*IA*是事件驱动（处理受限于网络包的大小以及数据到达的速率），不按时间切片去调度。  

    Only two types of external events are handled by the library's scheduler, because only these events can be detected by select(2) or poll(2): I/O events (a file descriptor is ready for I/O) and time events (some timeout has expired). However, other types of events (such as a signal sent to a process) can also be handled by converting them to I/O events. For example, a signal handling function can perform a write to a pipe (write(2) is reentrant/asynchronous-safe), thus converting a signal event to an I/O event.

只有两种外部事件会被库的调度器处理，因为它们能够被select(2)或者poll(2)捕获：I/O事件（I/O的fd准备就绪）和时间事件（一些超时被触发）。然而，其他的一些事件（比如有信号发送给进程）也能把它们转换为I/O事件。比如，一个信号处理函数能够执行管道的写入（write(2)可重入且也是异步安全的），因此转换信号事件为I/O事件。

    To take advantage of hardware parallelism, as in the EDSM architecture, multiple processes can be created in either a symmetric or asymmetric manner. Process management is not in the library's scope but instead is left up to the application. There are several general-purpose threading libraries that implement a many-to-one model (many user-level threads to one kernel execution vehicle), using the same basic techniques as the State Threads library (non-blocking I/O, event-driven scheduler, and so on). For an example, see GNU Portable Threads ([Reference 6]). Because they are general-purpose, these libraries have different objectives than the State Threads library. The State Threads library is not a general-purpose threading library, but rather an application library that targets only certain types of applications (IAs) in order to achieve the highest possible performance and scalability for those applications.

在*EDSM*中利用硬件的并行性，多进程能够创建对称或者不对称的方式。进程管理不在库的范围内，把它留给了应用程序。一些通用的线程库实现了多对一模型（多个用户态线程在一个*内核执行载体*上），State Threads库也利用相同的技术（非阻塞I/O，事件驱动调度器等等）。比如，GNU Protable Threads([Reference 6])。因为它们都是通用的，所以它们和State Threads库的目标不一样，State Threads库是一个应用库，目标是使得确定类型的*IA*获取最高的性能和柔性。

## 3.2 Scalability
## 3.2 可伸缩性

    State threads are very lightweight user-level entities, and therefore creating and maintaining user connections requires minimal resources. An application using the State Threads library scales very well with the increasing number of connections.

State threads是在用户态上，非常轻量，因此创建和维持用户连接需要非常小的资源。使用State Threads库的应用程序能够在连接数的增加时处理得非常好。

    On multiprocessor systems an application should create multiple processes to take advantage of hardware parallelism. Using multiple separate processes is the only way to achieve the highest possible system scalability. This is because duplicating per-process resources is the only way to avoid significant synchronization overhead on multiprocessor systems. Creating separate UNIX processes naturally offers resource duplication. Again, as in the EDSM architecture, there is no connection between the number of simultaneous connections (which may be very large and changes within a wide range) and the number of kernel entities (which is usually small and constant). In other words, the State Threads library makes it possible to multiplex a large number of simultaneous connections onto a much smaller number of separate processes, thus allowing an application to scale well with both the load and system size.

在多处理器系统中，一个应用程序应该利用硬件的并行性可多开启几个进程。使用多个独立的进程能够达到最高可能的系统可伸缩性。在多处理器系统中阻止数据同步开销的最有效的办法是复制每一个进程的资源。创建独立的UNIX进程原生提供资源复制。再次强调，在*EDSM*中，并发数（可能非常巨大，并且改变的范围很大）和*内核实体*（通常比较小）没有直接联系。换句话来说，State Threads库可以使得非常多的并发连接复用到较小的独立进程上，因此应用能够在负载可伸缩性和系统可伸缩性上处理得比较好。

## 3.3 Performance  
## 3.3 性能  

    Performance is one of the library's main objectives. The State Threads library is implemented to minimize the number of system calls and to make thread creation and context switching as fast as possible. For example, per-thread signal mask does not exist (unlike POSIX threads), so there is no need to save and restore a process's signal mask on every thread context switch. This eliminates two system calls per context switch. Signal events can be handled much more efficiently by converting them to I/O events (see above).

性能是这个库的主要目标之一。State Threads库的实现最小化系统调用的次数，并且使得线程的创建和上下文的切换尽可能的快。比如，每个线程的信号掩码都不存在（不像POSIX线程）。因此不需要在线程切换时保存和恢复信号掩码。每次上下文切换时减少了两次系统调用。信号事件通过转换成I/O事件处理起来更加有效率。

## 3.4 Portability
## 3.4 移植性

    The library uses the same general, underlying concepts as the EDSM architecture, including non-blocking I/O, file descriptors, and I/O multiplexing. These concepts are available in some form on most UNIX platforms, making the library very portable across many flavors of UNIX. There are only a few platform-dependent sections in the source.

库使用了跟*EDSM*相同的基本概念，包括非阻塞I/O，fd，和I/O复用。这些概念都以某种形态存在在大多数UNIX平台上，是得库在各种风格的UNIX上非常容易移植。在代码中只有极少平台相关特性。

## 3.5 State Threads and NSPR
## 3.5 State Threads库和NSPR

    The State Threads library is a derivative of the Netscape Portable Runtime library (NSPR) [Reference 7]. The primary goal of NSPR is to provide a platform-independent layer for system facilities, where system facilities include threads, thread synchronization, and I/O. Performance and scalability are not the main concern of NSPR. The State Threads library addresses performance and scalability while remaining much smaller than NSPR. It is contained in 8 source files as opposed to more than 400, but provides all the functionality that is needed to write efficient IAs on UNIX-like platforms.

State Threads库是从Netscape Portable Runtime library (NSPR) [Reference 7]发展而来。NSPR的最主要的目标是为系统工具提供一个平台无关层，系统工具包括线程、线程同步，以及I/O。性能和可伸缩性不是NSPR的主要关心的问题。State Threads库解决了性能和可伸缩性的问题，当时比NSPR小得多。它包含了8个源文件而不是NSPR的400个，却在类UNIX平台上提供了编写高效*IA*的所有功能。

NSPR	State Threads
Lines of code	~150,000	~3000
Dynamic library size  
(debug version)		
IRIX	~700 KB	~60 KB
Linux	~900 KB	~70 KB

# Conclusion
# 结论

    State Threads is an application library which provides a foundation for writing Internet Applications. To summarize, it has the following advantages:  
    
State Threads是一个提供了编写互联网应用程序的基础应用库。包含了以下优点：

    • It allows the design of fast and highly scalable applications. An application will scale well with both load and number of CPUs.  
    • It greatly simplifies application programming and debugging because, as a rule, no mutual exclusion locking is necessary and the entire application is free to use static variables and non-reentrant library functions.   
    
• 它允许设计快速和高可伸缩性的应用程序。应用程序能够在负载可伸缩性和系统伸缩性上同时处理得很好。  
• 由于没有互斥锁简化了编程和调试，整个应用可以随意的使用静态变量和不可重入的库函数。 

    The library's main limitation:   
     
库的主要限制：  

    • All I/O operations on sockets must use the State Thread library's I/O functions because only those functions perform thread scheduling and prevent the application's processes from blocking.    
• 所有在socket上的I/O操作必须使用State Thread库提供的I/O函数，因为只有这些函数执行线程调度时能够防止应用程序的进程阻塞。

###**References**

1.	Apache Software Foundation, http://www.apache.org.
2.	Douglas E. Comer, David L. Stevens, Internetworking With TCP/IP, Vol. III: Client-Server Programming And Applications, Second Edition, Ch. 8, 12.
3.	W. Richard Stevens, UNIX Network Programming, Second Edition, Vol. 1, Ch. 15.
4.	Zeus Technology Limited, http://www.zeus.co.uk.
5.	Peter Druschel, Vivek S. Pai, Willy Zwaenepoel, Flash: An Efficient and Portable Web Server. In Proceedings of the USENIX 1999 Annual Technical Conference, Monterey, CA, June 1999.
6.	GNU Portable Threads, http://www.gnu.org/software/pth/.
7.	Netscape Portable Runtime, http://www.mozilla.org/docs/refList/refNSPR/.
Other resources covering various architectural issues in IAs
8.	Dan Kegel, The C10K problem, http://www.kegel.com/c10k.html.
9.	James C. Hu, Douglas C. Schmidt, Irfan Pyarali, JAWS: Understanding High Performance Web Systems, http://www.cs.wustl.edu/~jxh/research/research.html.


# 学习总结 

`负载可伸缩性`: 它能够在较大负载（并发连接数）范围内维持它的吞吐量，如果负载依赖的系统资源越多（比如进程、线程），那么其负载可伸缩性越低。  
`系统可伸缩性`: 应用程序每单位硬件（比如CPU）能够增加的性能。如果任务之间的数据同步越多，那么其系统可伸缩性越低，因为很多时候都等待在锁上，物理资源没有得到充分利用。  
`代码可读性`: 指编写以及阅读代码的复杂程度，多进程架构符合人类线性思考模式，可读性最强；多线程架构增加了同步以及互斥，可读性稍差；EDSM是非阻塞，所有代码不是线性模式，来回切换，可读性最不好。

负载可伸缩性: EDSM > 多线程架构 > 多进程架构  
系统可伸缩性: EDSM > 多进程架构 > 多线程架构  
代码可读性： 多进程架构 > 多线程架构 > EDSM  

State Threads提出了一种既有高可伸缩性以及代码可读性较好的一种模式，实现了用户态的线程，也就是`协程`的概念。      
`协程`是个好东东，像同步程序一样写异步server，后续要多多学习下。